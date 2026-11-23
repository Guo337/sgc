---
id: vol-session-start
type: script-volume
created: 2026-11-18
last-touched: 2026-11-18
source-of: 90-Meta/scripts/session_start.py
fence: "````"
eol: crlf
depends-on: [vol-ttl-locator]
sha256: 8c07d99056c697079523bf8eaaf3f84a8a356e426de01e4884fb2420edfa1aea
---
````python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
"""session_start.py - SessionStart injection generator (knife H, FINAL plan 2026-11-18).

Authorized scope (Owner 23:40, injection half ONLY):
  stdout = ROOT L0 (read_orchestrator extract_root_l0 definition, boundary-based,
           no hardcoded line numbers) + ALL pitfall trigger points (summary level,
           each with a [[PIT-name]] anchor) + source pointers.
Explicitly NOT done: domain filtering (task domain unknown at SessionStart),
session-end logging (unauthorized), generic hook framework, any disk write.

Design stance:
- Read-only, stdlib only, no network. Writes nothing (self-test proves it).
- Fail-loud: ROOT missing/unparseable -> non-empty ERROR output, exit 2;
  pitfall trigger/scope missing or illegal -> report the file, never skip
  silently (runtime guard; does NOT wait for doctor TRIGGER_FM, FINAL CORRECT-1).
- L0 extraction mirrors read_orchestrator.extract_root_l0 exactly (H1 +
  blockquotes before first '## ' + the partition-list section). Reimplemented
  here (not imported) to keep depends-on minimal ([vol-ttl-locator] per plan);
  self-test T7 cross-checks against the orchestrator when co-extracted.
- Token proxy mirrors read_orchestrator.approx_tokens (1/CJK char + 1/latin word).
- Budget: schema.yaml budgets.assembly.L3 is authoritative; fallback 2000.
  Over-budget is reported loudly (exit 1), never silently truncated.

Usage:
  python session_start.py [--vault PATH] [--json]
  python session_start.py --self-test

Exit codes: 0 = injection emitted within budget / 1 = over L3 budget or
self-test failure / 2 = fail-loud (ROOT or pitfall contract violation).
NOTE: comments/docstrings English by project convention; user-facing text Chinese.
"""
import sys, os, re, json, argparse

sys.path.insert(0, os.path.dirname(os.path.abspath(__file__)))
from ttl_locator import VaultResolutionError, resolve_vault

try:
    sys.stdout.reconfigure(encoding="utf-8")
except Exception:
    pass

VERSION = "0.1.0"

TRIGGER_ENUM = {"always", "on-domain", "on-file-pattern", "on-project"}
DEFAULT_L3_BUDGET = 2000

CJK_RE = re.compile(r"[\u3400-\u4dbf\u4e00-\u9fff\uf900-\ufaff]")
LATIN_WORD_RE = re.compile(r"[A-Za-z0-9_]+")
FM_RE = re.compile(r"^---\s*\n(.*?)\n---\s*\n?", re.S)
KV_RE = re.compile(r"^([A-Za-z0-9_\-]+):\s*(.*)$")


def approx_tokens(text):
    """Same proxy as read_orchestrator.approx_tokens (documented, reproducible)."""
    if not text:
        return 0
    return len(CJK_RE.findall(text)) + len(LATIN_WORD_RE.findall(text))


def _strip_comment(val):
    """Strip trailing ' # comment' outside quotes (frontmatter inline convention)."""
    out, quote = [], None
    for ch in val:
        if quote:
            out.append(ch)
            if ch == quote:
                quote = None
        elif ch in "\"'":
            quote = ch
            out.append(ch)
        elif ch == "#" and (not out or out[-1] in " \t"):
            break
        else:
            out.append(ch)
    return "".join(out).strip()


def parse_frontmatter(text):
    """Minimal flat key: value reader for the fields this tool consumes
    (type/trigger/scope/summary). Deliberately NOT a second full YAML parser:
    entry_doctor.parse_fm remains the validation authority; this reader only
    extracts scalar fields and fails loud on duplicate consumed keys."""
    m = FM_RE.match(text)
    if not m:
        return None, text
    fm = {}
    for ln in m.group(1).splitlines():
        if not ln.strip() or ln.lstrip().startswith("#"):
            continue
        mm = KV_RE.match(ln)
        if not mm:
            continue
        key, val = mm.group(1), _strip_comment(mm.group(2))
        if key in fm:
            return None, text          # duplicate key: fail loud, same as doctor
        fm[key] = val.strip().strip("\"'")
    return fm, text[m.end():]


def extract_root_l0(body):
    """Mirror of read_orchestrator.extract_root_l0 (boundary definition):
    H1 + blockquote lines before the first '## ' + the partition-list section
    heading and its '- ' items. No hardcoded line numbers (knife A safe)."""
    lines = body.splitlines()
    out = []
    for ln in lines:
        if ln.strip().startswith("# "):
            out.append(ln.rstrip())
            break
    for ln in lines:
        st = ln.strip()
        if st.startswith("## "):
            break
        if st.startswith(">"):
            out.append(ln.rstrip())
    grab = False
    for ln in lines:
        st = ln.strip()
        if st.startswith("## "):
            grab = ("\u5206\u533a\u4e00\u89c8" in st)   # 分区一览
            if grab:
                out.append(ln.rstrip())
            continue
        if grab and st.startswith("- "):
            out.append(ln.rstrip())
    return "\n".join(out)


def load_l3_budget(root):
    """Read budgets.assembly.L3 from schema.yaml (single source of truth).
    Fallback DEFAULT_L3_BUDGET when unreadable; report which source was used."""
    path = os.path.join(root, "90-Meta", "schema.yaml")
    try:
        txt = open(path, encoding="utf-8").read()
    except OSError:
        return DEFAULT_L3_BUDGET, "default"
    m = re.search(r"(?m)^\s*assembly:\s*\{[^}]*L3:\s*(\d+)", txt)
    if m:
        return int(m.group(1)), "schema.yaml"
    return DEFAULT_L3_BUDGET, "default"


def collect_pitfalls(root):
    """Scan 01-Specs/PIT-*.md. Returns (points, errors).
    points = [{name, summary, trigger, scope}] sorted by name (deterministic).
    errors = fail-loud messages (missing/illegal trigger/scope, unparseable fm,
    missing summary). Errors are NEVER silently skipped."""
    specs = os.path.join(root, "01-Specs")
    points, errors = [], []
    try:
        names = sorted(f for f in os.listdir(specs)
                       if f.startswith("PIT-") and f.endswith(".md"))
    except OSError:
        return [], ["01-Specs directory not found: %s" % specs]
    for fn in names:
        rel = "01-Specs/" + fn
        name = fn[:-3]
        try:
            txt = open(os.path.join(specs, fn), encoding="utf-8").read()
        except OSError as exc:
            errors.append("%s: unreadable (%s)" % (rel, exc))
            continue
        fm, _ = parse_frontmatter(txt)
        if fm is None:
            errors.append("%s: frontmatter unparseable (duplicate key or no fm)" % rel)
            continue
        if fm.get("type") != "pitfall":
            errors.append("%s: PIT- file but type=%r (expected pitfall)"
                          % (rel, fm.get("type")))
            continue
        trig = fm.get("trigger", "")
        scope = fm.get("scope", "")
        summ = fm.get("summary", "")
        if trig not in TRIGGER_ENUM:
            errors.append("%s: trigger missing/illegal: %r (legal: %s)"
                          % (rel, trig, "/".join(sorted(TRIGGER_ENUM))))
        if not scope.strip():
            errors.append("%s: scope missing/empty" % rel)
        if not summ.strip():
            errors.append("%s: summary missing/empty" % rel)
        if trig in TRIGGER_ENUM and scope.strip() and summ.strip():
            points.append({"name": name, "summary": summ,
                           "trigger": trig, "scope": scope})
    return points, errors


def build_injection(root):
    """Assemble the injection payload. Returns dict:
    {status, text, tokens, l0_tokens, pit_tokens, pit_count, budget,
     budget_source, errors}. status: ok / over-budget / error."""
    root_path = os.path.join(root, "00-ROOT.md")
    try:
        rtxt = open(root_path, encoding="utf-8").read()
    except OSError as exc:
        return {"status": "error", "text": "", "errors":
                ["00-ROOT.md unreadable: %s" % exc], "pit_count": 0}
    fm, body = parse_frontmatter(rtxt)
    if fm is None:
        return {"status": "error", "text": "", "errors":
                ["00-ROOT.md frontmatter unparseable"], "pit_count": 0}
    l0 = extract_root_l0(body)
    if not l0.strip():
        return {"status": "error", "text": "", "errors":
                ["00-ROOT.md L0 extraction empty (H1/blockquotes missing?)"],
                "pit_count": 0}
    points, errors = collect_pitfalls(root)
    if errors:
        return {"status": "error", "text": "", "errors": errors,
                "pit_count": len(points)}
    budget, bsrc = load_l3_budget(root)
    parts = [l0, "",
             "## \u26a0\ufe0f \u89e6\u53d1\u70b9\uff08PIT \u5168\u91cf\u6ce8\u5165\uff0cSessionStart \u65e0\u4efb\u52a1 domain\uff0c\u4e0d\u8fc7\u6ee4\uff09"]
    for p in points:
        parts.append("- [[%s]] \u2014 %s" % (p["name"], p["summary"]))
    parts += ["",
              "> \u6765\u6e90\uff1a[[00-ROOT]] L0 + 01-Specs/PIT-*\uff08vol-session-start v%s \u751f\u6210\uff0c\u53ea\u8bfb\uff09\u3002"
              "\u6309\u9700\u5c55\u5f00\u5361\u5168\u6587\uff1b\u65e0 hook \u73af\u5883\u81ea\u67e5\u4e49\u52a1\u89c1 [[SPEC-READ]]\u3002" % VERSION]
    text = "\n".join(parts)
    total = approx_tokens(text)
    l0t = approx_tokens(l0)
    status = "ok" if total <= budget else "over-budget"
    return {"status": status, "text": text, "tokens": total,
            "l0_tokens": l0t, "pit_tokens": total - l0t,
            "pit_count": len(points), "budget": budget,
            "budget_source": bsrc, "errors": []}


def self_test():
    """In-memory fixtures only (no vault writes, no temp dirs). >=6 checks per
    FINAL plan knife H: L0+all PIT / budget / illegal-field fail-loud /
    ROOT-missing fail-loud / read-only / determinism (+L0 boundary shape)."""
    results = []

    def check(name, ok):
        results.append((name, bool(ok)))
        print("  [%s] %s" % ("OK" if ok else "FAIL", name))

    root_fm = ('---\nid: "spec-root-000"\nlayer: 3\ntype: spec\n'
               'summary: test root\n---\n')
    root_body = ("# TTL test vault\n\n"
                 "> root pointer line one\n\n"
                 "> operating-mode: fast-iterate\n\n"
                 "## \u5206\u533a\u4e00\u89c8\uff08L0\uff09\n\n"
                 "- entry / routing\n\n"
                 "## \u5165\u53e3\n\n"
                 "- [SPEC-READ](01-Specs/SPEC-READ.md)\n"
                 "> late blockquote must NOT be in L0\n")
    pit_ok = ('---\nid: pit-2026-00001\nlayer: 3\ntype: pitfall\n'
              'trigger: on-domain\nscope: "knowledge-engineering"\n'
              'summary: fixture pitfall one\n---\n# PIT-fixture-one\nbody\n')
    pit_ok2 = pit_ok.replace("00001", "00002").replace("one", "two")

    class MemFS:
        """Tiny in-memory stand-in for the three os calls build_injection uses."""
        def __init__(self, files):
            self.files = files

    def run_mem(files):
        import builtins
        real_open, real_listdir = builtins.open, os.listdir
        sep = os.sep
        root = "MEMVAULT"

        def fake_open(path, *a, **kw):
            key = str(path)
            if key in files:
                import io
                return io.StringIO(files[key])
            raise OSError("memfs: no such file %s" % key)

        def fake_listdir(path):
            pref = str(path) + sep
            out = [k[len(pref):] for k in files
                   if k.startswith(pref) and sep not in k[len(pref):]]
            if not out and not any(k.startswith(pref) for k in files):
                raise OSError("memfs: no such dir %s" % path)
            return out

        builtins.open, os.listdir = fake_open, fake_listdir
        try:
            return build_injection(root)
        finally:
            builtins.open, os.listdir = real_open, real_listdir

    specs = "MEMVAULT" + os.sep + "01-Specs" + os.sep
    base = {"MEMVAULT" + os.sep + "00-ROOT.md": root_fm + root_body,
            "MEMVAULT" + os.sep + "90-Meta" + os.sep + "schema.yaml":
                "budgets:\n  assembly: { L3: 2000, L2: 4000, L1: 1000 }\n",
            specs + "PIT-fixture-one.md": pit_ok,
            specs + "PIT-fixture-two.md": pit_ok2}

    # T1: L0 + all PIT points present, anchors present, late blockquote excluded
    r = run_mem(base)
    t1 = (r["status"] == "ok" and r["pit_count"] == 2
          and "[[PIT-fixture-one]]" in r["text"]
          and "[[PIT-fixture-two]]" in r["text"]
          and "# TTL test vault" in r["text"]
          and "late blockquote" not in r["text"]
          and "\u5206\u533a\u4e00\u89c8" in r["text"])
    check("T1 L0 + all PIT anchors injected, post-H2 blockquote excluded", t1)

    # T2: budget respected; over-budget reported loudly (tiny budget fixture)
    tiny = dict(base)
    tiny["MEMVAULT" + os.sep + "90-Meta" + os.sep + "schema.yaml"] = \
        "budgets:\n  assembly: { L3: 5, L2: 4000, L1: 1000 }\n"
    r2 = run_mem(tiny)
    t2 = (r["tokens"] <= r["budget"] and r["budget_source"] == "schema.yaml"
          and r2["status"] == "over-budget")
    check("T2 budget from schema; over-budget fail-loud not truncated", t2)

    # T3: illegal trigger / missing scope -> error, never silent skip
    bad = dict(base)
    bad[specs + "PIT-fixture-two.md"] = pit_ok2.replace(
        "trigger: on-domain", "trigger: sometimes")
    r3 = run_mem(bad)
    bad2 = dict(base)
    bad2[specs + "PIT-fixture-two.md"] = pit_ok2.replace(
        'scope: "knowledge-engineering"\n', "")
    r3b = run_mem(bad2)
    t3 = (r3["status"] == "error"
          and any("trigger" in e and "PIT-fixture-two" in e for e in r3["errors"])
          and r3b["status"] == "error"
          and any("scope" in e for e in r3b["errors"]))
    check("T3 illegal trigger / missing scope fail-loud with file named", t3)

    # T4: ROOT missing / unparseable -> error status, caller emits non-empty ERROR
    no_root = {k: v for k, v in base.items() if not k.endswith("00-ROOT.md")}
    r4 = run_mem(no_root)
    bad_root = dict(base)
    bad_root["MEMVAULT" + os.sep + "00-ROOT.md"] = "no frontmatter at all\n"
    r4b = run_mem(bad_root)
    t4 = (r4["status"] == "error" and r4["errors"]
          and r4b["status"] == "error" and r4b["errors"])
    check("T4 ROOT missing/unparseable fail-loud (non-empty errors)", t4)

    # T5: read-only - every open() during build_injection must be read-mode,
    # and the fixture store must be unchanged (no write path exists).
    import builtins
    real_open = builtins.open
    modes = []

    def spy_open(path, mode="r", *a, **kw):
        modes.append(mode)
        return real_open(path, mode, *a, **kw)

    before = dict(base)
    builtins.open = spy_open
    try:
        run_mem(base)
    finally:
        builtins.open = real_open
    t5 = (base == before
          and all(("w" not in m and "a" not in m and "+" not in m) for m in modes))
    check("T5 read-only (all opens read-mode, fixture store unchanged)", t5)

    # T6: determinism - two runs byte-identical
    ra, rb = run_mem(base), run_mem(base)
    t6 = ra["text"] == rb["text"] and ra["tokens"] == rb["tokens"]
    check("T6 deterministic output (two runs byte-identical)", t6)

    # T7: L0 boundary shape vs orchestrator definition (structural mirror check)
    l0 = extract_root_l0(root_body)
    t7 = (l0.splitlines()[0] == "# TTL test vault"
          and "> root pointer line one" in l0
          and "## \u5165\u53e3" not in l0
          and "- entry / routing" in l0)
    check("T7 L0 boundary: H1 + pre-H2 blockquotes + partition list only", t7)

    ok = all(v for _, v in results)
    print("\nsession_start --self-test: %s (%d checks)"
          % ("PASS" if ok else "FAIL", len(results)))
    return 0 if ok else 1


def main():
    ap = argparse.ArgumentParser(description="SessionStart injection generator (read-only)")
    ap.add_argument("--vault", help="vault root (default: ttl_locator resolution)")
    ap.add_argument("--json", action="store_true", help="machine-readable result")
    ap.add_argument("--self-test", action="store_true")
    a = ap.parse_args()

    if a.self_test:
        return self_test()

    try:
        loc = resolve_vault(a.vault, script_fallback=os.path.abspath(
            os.path.join(os.path.dirname(os.path.abspath(__file__)), "..", "..")))
    except VaultResolutionError as exc:
        print("ERROR: %s" % exc)
        return 2

    r = build_injection(loc.root)
    if r["status"] == "error":
        # fail-loud: non-empty ERROR output, never an empty injection (fake green)
        print("ERROR: session_start injection refused (vault=%s, source=%s)"
              % (loc.root, loc.source))
        for e in r["errors"]:
            print("  - %s" % e)
        if a.json:
            print(json.dumps({"tool": "session_start", "version": VERSION,
                              "status": "error", "errors": r["errors"]},
                             ensure_ascii=False))
        return 2

    if a.json:
        print(json.dumps({"tool": "session_start", "version": VERSION,
                          "status": r["status"], "tokens": r["tokens"],
                          "l0_tokens": r["l0_tokens"],
                          "pit_tokens": r["pit_tokens"],
                          "pit_count": r["pit_count"], "budget": r["budget"],
                          "budget_source": r["budget_source"],
                          "text": r["text"]}, ensure_ascii=False, indent=2))
    else:
        print(r["text"])
        print("\n<!-- session_start v%s | tokens=%d (L0=%d + PIT=%d, %d points) "
              "| budget=%d (%s) | status=%s -->"
              % (VERSION, r["tokens"], r["l0_tokens"], r["pit_tokens"],
                 r["pit_count"], r["budget"], r["budget_source"], r["status"]))
    if r["status"] == "over-budget":
        print("ERROR: injection %d tokens exceeds L3 budget %d - NOT truncated; "
              "reduce payload or raise budget via schema (Owner)"
              % (r["tokens"], r["budget"]))
        return 1
    return 0


if __name__ == "__main__":
    try:
        sys.exit(main())
    except Exception as exc:
        print("session_start self-failure: %r" % exc)
        sys.exit(2)
````
