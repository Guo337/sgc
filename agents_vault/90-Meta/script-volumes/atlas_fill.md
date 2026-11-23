---
id: vol-atlas-fill
type: script-volume
created: 2026-11-20
last-touched: 2026-11-20
source-of: 90-Meta/scripts/atlas_fill.py
fence: "````"
eol: crlf
depends-on: [vol-ttl-locator, vol-entry-doctor]
sha256: 540529fb22a4a5025b0b09fc261150a2395eb9baf02c8f9d4878e8f5f1e4bf60
---
````python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
"""atlas_fill.py — SH auto-block projection generator (knife G, nav wave 2/3).

Authorized scope (Owner 2026-11-18 23:40 + PLAN-nav-wave23-closure r1, wave A):
  Regenerate the atlas:auto block of the knowledge-engineering SH from the four
  derivation rules whose authoritative text lives in the block's leading comment
  (D32/corr-77):
    (1) C-1 membership = 02-Cards cards whose domain contains knowledge-engineering
        (recursive, incubator included);
    (2) grouping = hub.abstracts edges (multi-membership cards listed in EVERY
        group); instance cards with no hub membership go to the unclustered
        group (hub:true cards themselves head their groups and are NOT listed
        unclustered — Owner ruling 2A, corr-2026-11-20-93, G principle: the
        exemption carries the computable hub:true criterion);
    (3) federation copy list = block list minus 02-Cards/incubator/ path cards
        (stdout comparison table only; the guide card is never written);
    (4) acceptance = regenerated line set == current block line set (diff = empty).
Explicitly NOT done: MAP.yaml (stays GHOST), multi-SH routing, domain filtering,
hooks (D32 veto), any write outside the auto-start/end markers.

Design stance:
- The leading comment lines are hand-maintained authority text: they are
  extracted from the CURRENT block and preserved verbatim, never hardcoded.
- Default is dry-run (stdout proposed block + unified diff vs current block);
  --apply rewrites ONLY the bytes between the auto markers. Write channel:
  official obsidian CLI first (append_log conventions: resolved full-path
  launcher, active-vault pin, backslash/size gates); explicit downgrade to an
  offline atomic write + byte-exact read-back verification when unavailable.
- Fail-loud: SH missing / auto markers missing / hub abstracts edge pointing at
  a nonexistent card -> non-zero exit, never silent skip. Over the SH auto soft
  budget (400 tok, approx_tokens proxy: 1/CJK char + 1/latin word, same as
  read_orchestrator/session_start) -> loud report, exit 1, no silent truncation.
- Frontmatter parsing reuses entry_doctor.parse_fm (no second parser;
  depends-on: vol-entry-doctor, vol-ttl-locator).
- Stdlib only; Windows-safe (explicit encoding="utf-8", newline="" reads).

Usage:
  python atlas_fill.py [--vault PATH] [--channel cli|offline] [--apply]
  python atlas_fill.py --self-test

Exit codes: 0 ok / 1 diff non-empty or over soft budget / 2 fail-loud contract
violation or tool self-failure.
NOTE: comments/docstrings English by project convention; user-facing text Chinese.
"""
import sys, os, re, argparse, difflib, hashlib, tempfile

# Reuse the doctor's battle-tested frontmatter parser (no drift, no second parser).
# depends-on: vol-entry-doctor, vol-ttl-locator (extract both before running)
sys.path.insert(0, os.path.dirname(os.path.abspath(__file__)))
from entry_doctor import parse_fm, as_name
from ttl_locator import VaultResolutionError, resolve_vault

try:
    sys.stdout.reconfigure(encoding="utf-8")
except Exception:
    pass

VERSION = "0.1.0"

DOMAIN = "knowledge-engineering"
SH_REL = "03-Atlas/knowledge-engineering/SH-knowledge-engineering.md"
CARDS_PREFIX = "02-Cards/"
INCUBATOR_PREFIX = "02-Cards/incubator/"
AUTO_SOFT_BUDGET = 400          # SH auto soft budget (tok proxy), nav plan 2026-11-18

AUTO_START = "<!-- atlas:auto-start -->"
AUTO_END = "<!-- atlas:auto-end -->"

CJK_RE = re.compile(r"[\u3400-\u4dbf\u4e00-\u9fff\uf900-\ufaff]")
LATIN_WORD_RE = re.compile(r"[A-Za-z0-9_]+")
EDGE_TARGET_RE = re.compile(r"target:\s*\"?\[\[([^\]\|#]+)")
EDGE_REL_RE = re.compile(r"rel:\s*([a-z_]+)")
LIST_LINE_RE = re.compile(r"^\s*-\s+\[\[([^\]\|#]+)")


class AtlasFillError(RuntimeError):
    """A fail-loud contract violation (missing SH/markers, broken abstracts edge)."""


def approx_tokens(text):
    """Same proxy as read_orchestrator/session_start (documented, reproducible):
    1 token per CJK char plus 1 token per latin/digit word."""
    if not text:
        return 0
    return len(CJK_RE.findall(text)) + len(LATIN_WORD_RE.findall(text))


def read_text(path):
    """Byte-faithful read: newline='' keeps the file's actual CRLF/LF endings."""
    with open(path, encoding="utf-8", newline="") as fh:
        return fh.read()


def load_cards(root):
    """Walk 02-Cards recursively (incubator included). Returns {rel: fm} for
    frontmatter-bearing cards; unparseable cards are skipped here (doctor
    FM_MISSING owns that report, the generator is not a second validator)."""
    cards = {}
    base = os.path.join(root, "02-Cards")
    if not os.path.isdir(base):
        raise AtlasFillError("02-Cards 目录不存在: %s" % base)
    for dp, dn, fn in os.walk(base):
        dn[:] = [d for d in dn if d not in ("__pycache__", ".obsidian", ".trash")]
        for f in sorted(fn):
            if not f.endswith(".md"):
                continue
            full = os.path.join(dp, f)
            rel = os.path.relpath(full, root).replace("\\", "/")
            fm, _ = parse_fm(read_text(full))
            if isinstance(fm, dict):
                cards[rel] = fm
    return cards


def domain_has(fm, domain):
    dom = fm.get("domain")
    if isinstance(dom, list):
        return domain in [str(d).strip() for d in dom]
    if isinstance(dom, str):
        return domain in [d.strip() for d in dom.strip("[]").split(",")]
    return False


def collect_members(cards):
    """Rule (1) C-1: name -> rel for 02-Cards cards whose domain contains DOMAIN."""
    members = {}
    for rel in sorted(cards):
        if domain_has(cards[rel], DOMAIN):
            members[as_name(rel)] = rel
    return members


def abstracts_targets(fm):
    """abstracts edge targets of one card, in frontmatter declaration order."""
    out = []
    ev = fm.get("e")
    if isinstance(ev, list):
        for item in ev:
            if not isinstance(item, str):
                continue
            mt = EDGE_TARGET_RE.search(item)
            mr = EDGE_REL_RE.search(item)
            if mt and mr and mr.group(1) == "abstracts":
                out.append(mt.group(1).strip().split("/")[-1])
    return out


def split_auto_block(sh_text):
    """Locate the auto block. Returns (prefix, inner, suffix) where prefix ends
    right after the start marker and suffix begins at the end marker; the
    generator may only ever replace `inner`. Fail-loud when markers missing."""
    i = sh_text.find(AUTO_START)
    j = sh_text.find(AUTO_END)
    if i < 0 or j < 0 or j < i:
        raise AtlasFillError("SH auto 标记缺失或次序颠倒（atlas:auto-start/end）")
    if sh_text.find(AUTO_START, i + 1) >= 0 or sh_text.find(AUTO_END, j + 1) >= 0:
        raise AtlasFillError("SH auto 标记重复（须恰一对 start/end）")
    return (sh_text[:i + len(AUTO_START)],
            sh_text[i + len(AUTO_START):j],
            sh_text[j:])


def leading_comment_lines(inner):
    """The hand-maintained derivation-rule comment: the leading run of non-list
    lines (blank lines included) before the first '- [[' list line. Preserved
    verbatim — the rule text is authority, the generator only makes list lines.
    Returned WITHOUT line endings; generate_block re-joins with the block eol."""
    lines = inner.split("\n")
    out = []
    for ln in lines:
        ln = ln.rstrip("\r")
        if LIST_LINE_RE.match(ln):
            break
        out.append(ln)
    return out


def generate_block(cards, members, comment_lines, eol, prev_inner=None):
    """Rules (1)+(2): hub groups in hub-name order, members in abstracts-edge
    declaration order (multi-membership listed in every group). Unclustered =
    C-1 minus hub-group union minus hub:true cards (2A, corr-93); its ORDER is
    not fixed by the rules — existing members keep their current relative
    order (the manual list is the acceptance baseline), new members are
    appended in name order. Returns (inner_text, groups, unclustered)."""
    hubs = sorted((members[n], n) for n in members
                  if cards[members[n]].get("hub") is True)
    grouped = set()
    groups = []          # [(hub_name, [member names])]
    for rel, name in hubs:
        targets = abstracts_targets(cards[rel])
        listed = []
        for tgt in targets:
            if tgt not in members:
                # fail-loud: an abstracts edge to a nonexistent card is a broken
                # edge; an edge to an existing non-C-1 card is a rule conflict —
                # both stop the run, never silently dropped (PLAN knife G).
                exists = any(as_name(r) == tgt for r in cards)
                why = "目标卡不在 C-1 收录集（domain 不含 %s）" % DOMAIN if exists \
                    else "目标卡不存在（断链）"
                raise AtlasFillError("枢纽 [[%s]] abstracts→[[%s]]：%s"
                                     % (name, tgt, why))
            if tgt not in listed:
                listed.append(tgt)
        grouped.update(listed)
        groups.append((name, listed))
    # Rule (2) as amended by Owner ruling 2A (corr-2026-11-20-93): unclustered =
    # C-1 set minus all hub-group members minus hub:true cards themselves (hubs
    # head their groups; listing them unclustered would duplicate the head).
    hub_names = {name for _, name in hubs}
    unc_set = {n for n in members if n not in grouped and n not in hub_names}
    prev_order = []
    if prev_inner:
        # keep the current unclustered relative order for surviving members
        in_unc = False
        for ln in prev_inner.split("\n"):
            ln = ln.rstrip("\r")
            if re.match(r"^-\s+未聚簇", ln):
                in_unc = True
                continue
            if in_unc:
                m = LIST_LINE_RE.match(ln)
                if m and m.group(1) in unc_set and m.group(1) not in prev_order:
                    prev_order.append(m.group(1))
    unclustered = prev_order + sorted(unc_set - set(prev_order))
    lines = list(comment_lines)
    for name, listed in groups:
        lines.append("- [[%s]]（枢纽）" % name)
        lines.extend("  - [[%s]]" % t for t in listed)
    lines.append("- 未聚簇：")
    lines.extend("  - [[%s]]" % t for t in unclustered)
    inner = eol.join(lines) + eol
    return inner, groups, unclustered


def federation_table(members):
    """Rule (3): block list minus incubator-path cards. Stdout comparison table
    only — the guide card ([[副库建库引导]]) is never written by this tool."""
    rows = []
    for name in sorted(members):
        rel = members[name]
        inc = rel.startswith(INCUBATOR_PREFIX)
        rows.append((name, rel, "排除" if inc else "复制"))
    copy = [n for n, r, s in rows if s == "复制"]
    excluded = [n for n, r, s in rows if s != "复制"]
    return rows, copy, excluded


def unified_diff(old_inner, new_inner, rel):
    return "".join(difflib.unified_diff(
        old_inner.splitlines(keepends=True), new_inner.splitlines(keepends=True),
        fromfile="a/" + rel + " (现区块)", tofile="b/" + rel + " (拟生成)"))


# --------------------------------------------------------------------------- #
# write channels (append_log conventions)
# --------------------------------------------------------------------------- #
def atomic_write(path, text):
    """Offline atomic write: temp file in same dir + os.replace, then the caller
    verifies read-back bytes. newline='' keeps the intended CRLF/LF verbatim."""
    d = os.path.dirname(os.path.abspath(path)) or "."
    fd, tmp = tempfile.mkstemp(prefix=".atlas_fill.", suffix=".tmp", dir=d)
    try:
        with os.fdopen(fd, "w", encoding="utf-8", newline="") as f:
            f.write(text)
        os.replace(tmp, path)
    except BaseException:
        if os.path.exists(tmp):
            os.unlink(tmp)
        raise


def _resolve_obsidian():
    """Resolve the CLI launcher to a FULL PATH (append_log convention): the bare
    name would launch the GUI Obsidian.exe on Windows; the real CLI is the small
    console launcher Obsidian.com (PIT-obsidian裸名误调GUI版)."""
    import shutil
    p = shutil.which("obsidian")
    if not p:
        return None
    if p.lower().endswith(".exe"):
        com = p[:-4] + ".com"
        if os.path.exists(com):
            return com
    return p


def _active_vault_matches(obs, vault):
    """(ok, info): the App's ACTIVE vault must be our target vault, else the CLI
    write path is unavailable and the caller downgrades explicitly."""
    import subprocess
    try:
        r = subprocess.run([obs, "vault", "info=path"], capture_output=True,
                           text=True, encoding="utf-8", timeout=20)
    except Exception as e:
        return False, "vault 查询失败: %s" % e
    active = (r.stdout or "").strip()
    if r.returncode != 0 or not active:
        return False, "App 不可达（vault info rc=%s）" % r.returncode
    if os.path.normcase(os.path.normpath(active)) != os.path.normcase(os.path.normpath(vault)):
        return False, "活动 vault %r != 目标 %r" % (active, vault)
    return True, active


# Conservative payload cap (append_log precedent): large payloads corrupt the
# CLI->App IPC JSON; the true safe limit is unknown, so anything above the cap
# downgrades to offline instead of probing a live App.
CLI_MAX_BYTES = 2048


def _escape_cli_content(text):
    """Encode real newlines/tabs as literal \\n / \\t (append_log convention).
    Callers must gate on the absence of literal backslashes first."""
    return text.replace("\n", "\\n").replace("\t", "\\t")


def _assert_only_block_changed(old_text, new_text):
    """The write must touch ONLY the auto block: prefix/suffix byte-identical."""
    op, oi, os_ = split_auto_block(old_text)
    np, ni, ns = split_auto_block(new_text)
    if op != np or os_ != ns:
        raise AtlasFillError("区块外字节发生变化（违反仅自动生成区写权限）")


def write_sh(vault, rel, old_text, new_text, channel):
    """--apply write path. CLI first (official channel), explicit downgrade to
    offline atomic write; both followed by byte-exact read-back verification."""
    abs_path = os.path.join(vault, rel.replace("/", os.sep))
    if channel == "cli":
        reason = None
        obs = _resolve_obsidian()
        nbytes = len(new_text.encode("utf-8"))
        if "\\" in new_text:
            reason = "内容含反斜杠，CLI/IPC 会损坏"
        elif nbytes > CLI_MAX_BYTES:
            reason = "内容 %d 字节 > CLI_MAX_BYTES %d（大载荷会损坏 CLI→App IPC）" % (nbytes, CLI_MAX_BYTES)
        elif obs is None:
            reason = "PATH 上未找到 obsidian CLI"
        else:
            ok, info = _active_vault_matches(obs, vault)
            if not ok:
                reason = info
        if reason is not None:
            print("[cli] %s —— 显式降级 offline 原子写" % reason)
        else:
            import subprocess
            args = [obs, "create", "path=" + rel,
                    "content=" + _escape_cli_content(new_text), "overwrite"]
            r = subprocess.run(args, capture_output=True, text=True,
                               encoding="utf-8", cwd=vault)
            if r.returncode != 0:
                raise AtlasFillError("obsidian create 失败 (rc=%s): %s"
                                     % (r.returncode, (r.stderr or r.stdout or "").strip()[:200]))
            got = read_text(abs_path)
            if got != new_text:
                raise AtlasFillError("CLI 写回读校验失败：磁盘字节 != 拟写入（用 --channel offline 重试）")
            _assert_only_block_changed(old_text, got)
            print("[cli] 写入完成，回读逐字节校验通过")
            return
    atomic_write(abs_path, new_text)
    got = read_text(abs_path)
    if got != new_text:
        raise AtlasFillError("offline 写回读校验失败：磁盘字节 != 拟写入")
    _assert_only_block_changed(old_text, got)
    print("[offline] 原子写完成，回读逐字节校验通过")


# --------------------------------------------------------------------------- #
# run
# --------------------------------------------------------------------------- #
def run(root, apply=False, channel="cli", sh_rel=SH_REL, quiet=False):
    """One projection run. Returns exit code; prints the proposed block, diff,
    federation table and budget report (Chinese, user-facing)."""
    sh_path = os.path.join(root, sh_rel.replace("/", os.sep))
    if not os.path.isfile(sh_path):
        raise AtlasFillError("SH 缺失: %s" % sh_rel)
    sh_text = read_text(sh_path)
    prefix, inner, suffix = split_auto_block(sh_text)
    eol = "\r\n" if "\r\n" in inner else "\n"
    comment = leading_comment_lines(inner)
    cards = load_cards(root)
    members = collect_members(cards)
    new_inner, groups, unclustered = generate_block(cards, members, comment, eol,
                                                    prev_inner=inner)
    diff = unified_diff(inner, new_inner, sh_rel)
    # Budget scope: the GENERATED projection (list lines) is what the 400-tok
    # soft budget governs (nav plan: "投影层（auto 区块，G 机械生成，≤400 tok）").
    # The hand-maintained rule comment is authority text, not generated payload;
    # its cost is reported separately for transparency.
    tokens = approx_tokens(eol.join(new_inner.split(eol)[len(comment):]))
    comment_tokens = approx_tokens(eol.join(comment))
    rows, copy, excluded = federation_table(members)

    if not quiet:
        print("atlas_fill v%s：%s（C-1 收录 %d 张；枢纽组 %d；未聚簇 %d）"
              % (VERSION, sh_rel, len(members), len(groups), len(unclustered)))
        print("\n[拟生成 auto 区块]")
        sys.stdout.write(new_inner)
        print("\n[与现区块 diff]")
        if diff:
            sys.stdout.write(diff)
        else:
            print("diff=∅（生成内容与现区块逐字节一致）")
        print("\n[联邦名单对照表]（规则③：清单 − incubator 路径；仅 stdout，不写引导卡）")
        for name, rel, status in rows:
            print("  %s  %s  ← %s" % (status, name, rel))
        print("联邦复制 %d 张 / incubator 排除 %d 张" % (len(copy), len(excluded)))
        print("\n[预算] 生成清单 approx_tokens=%d / 软预算 %d：%s（另：手工规则注释 %d tok，权威文本不计入生成预算；区块合计 %d）"
              % (tokens, AUTO_SOFT_BUDGET,
                 "超预算（响亮报告，不静默截断）" if tokens > AUTO_SOFT_BUDGET else "在预算内",
                 comment_tokens, tokens + comment_tokens))

    if apply:
        # --apply is the sanctioned regeneration path (SPEC-WRITE matrix: auto
        # block only). The diff is printed above for attribution; the first-run
        # diff=∅ acceptance bar is the OPERATOR's gate (run dry-run first), not
        # an unconditional refusal here — otherwise the tool could never repair
        # a drifted block, which is its whole purpose.
        new_text = prefix + new_inner + suffix
        if new_text == sh_text:
            if not quiet:
                print("[apply] 无字节变化，跳过写入")
        else:
            write_sh(root, sh_rel, sh_text, new_text, channel)
    if tokens > AUTO_SOFT_BUDGET:
        return 1
    return 1 if diff else 0


# --------------------------------------------------------------------------- #
# self-test (offline fixtures in a scratch dir under cwd; never the real vault)
# --------------------------------------------------------------------------- #
def _card(name, domain="[knowledge-engineering]", hub=False, abstracts=(), seq=1):
    """Minimal KE card fixture (frontmatter only; the generator reads fm only)."""
    lines = ['---', 'id: "c-fix-%05d"' % seq, 'layer: 2', 'type: concept',
             'project: memory-lib', 'domain: %s' % domain, 'maturity: seed',
             'created: 2026-11-20', 'last-touched: 2026-11-20',
             'summary: fixture %s' % name]
    if hub:
        lines.append('hub: true')
    if abstracts:
        lines.append('e:')
        for t in abstracts:
            lines.append('  - { target: "[[%s]]", rel: abstracts, weight: 0.7 }' % t)
    lines += ['---', '', '# %s' % name, '']
    return "\n".join(lines)


def _sh(comment, listing, eol="\n"):
    body = eol.join([AUTO_START] + comment + listing + [AUTO_END])
    return eol.join(['---', 'id: "s-fix-sh"', 'layer: 2', 'type: summary',
                     'project: memory-lib', 'domain: [knowledge-engineering]',
                     'maturity: seed', 'created: 2026-11-20',
                     'last-touched: 2026-11-20', 'summary: fixture SH',
                     'edge-policy: exempt', '---', '', '# SH-fixture', '',
                     '## 卡片清单', '', body, '', '## 域间桥', '', '- 桥行', ''])


def self_test():
    """Offline fixtures (scratch dir under cwd, removed afterwards; the real
    vault is never touched). >=6 checks per PLAN knife G."""
    import shutil
    results = []

    def check(name, ok):
        results.append((name, bool(ok)))
        print("  [%s] %s" % ("OK" if ok else "FAIL", name))

    tmp = tempfile.mkdtemp(prefix=".atlas_fill_selftest.", dir=os.getcwd())
    try:
        comment = ["（派生规则·fixture：①C-1 ②hub.abstracts ③联邦−incubator ④diff=∅）"]
        # fixture population: hubA abstracts 甲+乙; hubB abstracts 乙+丙 (乙 is
        # multi-membership); 丁 unclustered; incubator 戊 stays listed but is
        # excluded from federation; 域外 card must not enter C-1.
        listing = ["- [[hubA]]（枢纽）", "  - [[甲]]", "  - [[乙]]",
                   "- [[hubB]]（枢纽）", "  - [[乙]]", "  - [[丙]]",
                   "- 未聚簇：", "  - [[丁]]", "  - [[戊]]"]
        cards = {
            "02-Cards/hubA.md": _card("hubA", hub=True, abstracts=("甲", "乙"), seq=1),
            "02-Cards/hubB.md": _card("hubB", hub=True, abstracts=("乙", "丙"), seq=2),
            "02-Cards/甲.md": _card("甲", seq=3),
            "02-Cards/乙.md": _card("乙", seq=4),
            "02-Cards/丙.md": _card("丙", seq=5),
            "02-Cards/丁.md": _card("丁", seq=6),
            "02-Cards/incubator/戊.md": _card("戊", seq=7),
            "02-Cards/域外.md": _card("域外", domain="[retrieval]", seq=8),
        }

        def build_vault(name, sh_text, extra=None):
            root = os.path.join(tmp, name)
            merged = dict(cards)
            merged.update(extra or {})
            for rel, txt in merged.items():
                p = os.path.join(root, rel.replace("/", os.sep))
                os.makedirs(os.path.dirname(p), exist_ok=True)
                with open(p, "w", encoding="utf-8", newline="") as f:
                    f.write(txt)
            p = os.path.join(root, SH_REL.replace("/", os.sep))
            os.makedirs(os.path.dirname(p), exist_ok=True)
            with open(p, "w", encoding="utf-8", newline="") as f:
                f.write(sh_text)
            return root

        def snapshot(root):
            out = {}
            for dp, dn, fn in os.walk(root):
                for f in fn:
                    p = os.path.join(dp, f)
                    with open(p, "rb") as fh:
                        out[p] = hashlib.sha256(fh.read()).hexdigest()
            return out

        import io
        from contextlib import redirect_stdout

        def run_quiet(root, **kw):
            buf = io.StringIO()
            with redirect_stdout(buf):
                rc = run(root, quiet=False, **kw)
            return rc, buf.getvalue()

        # T1: clean fixture — generated block == expected manual listing, diff=∅
        root1 = build_vault("t1", _sh(comment, listing))
        rc1, _ = run_quiet(root1)
        _, inner1, _ = split_auto_block(read_text(os.path.join(root1, SH_REL.replace("/", os.sep))))
        expected_inner = "\n" + "\n".join(comment + listing) + "\n"
        check("T1 干净 fixture 生成==期望，diff=∅（rc=0）", rc1 == 0 and inner1 == expected_inner)

        # T2: multi-membership card listed in EVERY hub group
        cards2 = load_cards(root1)
        members2 = collect_members(cards2)
        _, groups2, unc2 = generate_block(cards2, members2, comment, "\n")
        gmap = dict(groups2)
        check("T2 多归属卡每组都列（乙∈hubA∧hubB，不入未聚簇；域外卡不入 C-1）",
              "乙" in gmap.get("hubA", []) and "乙" in gmap.get("hubB", [])
              and "乙" not in unc2 and "域外" not in members2)

        # T2b: hub cards themselves never enter the unclustered group (2A,
        # corr-93): hubA/hubB are in C-1 but head their groups only.
        check("T2b hub 卡自身不入未聚簇（2A 豁免：hubA/hubB 只作组头）",
              "hubA" in members2 and "hubB" in members2
              and "hubA" not in unc2 and "hubB" not in unc2
              and unc2 == sorted(["丁", "戊"]))

        # T3: incubator card stays in the block list but is federation-excluded
        rows3, copy3, excl3 = federation_table(members2)
        listed_names = [m for ln in expected_inner.split("\n")
                        for m in LIST_LINE_RE.findall(ln)]
        check("T3 incubator 留清单但联邦排除（戊在清单/排除名单，不在复制名单）",
              "戊" in listed_names and "戊" in excl3 and "戊" not in copy3
              and len(copy3) == len(members2) - 1)

        # T4: auto markers missing -> fail-loud (named error, never silent)
        root4 = build_vault("t4", _sh(comment, listing).replace(AUTO_START, "<!-- 无标记 -->"))
        try:
            run_quiet(root4)
            t4 = False
        except AtlasFillError as e:
            t4 = "auto 标记" in str(e)
        check("T4 auto 标记缺失 fail-loud（AtlasFillError，非静默）", t4)

        # T5: --apply touches ONLY the block (outside bytes byte-identical).
        # Seed a stale block (丁 misplaced into hubA group) so apply performs a
        # real write while the unclustered order stays recoverable from prev.
        stale = [ln for ln in listing if ln != "  - [[丁]]"]
        stale.insert(3, "  - [[丁]]")
        root5 = build_vault("t5", _sh(comment, stale))
        sh5 = os.path.join(root5, SH_REL.replace("/", os.sep))
        before5 = read_text(sh5)
        bp, bi, bs = split_auto_block(before5)
        snap_before = snapshot(root5)
        rc5, _ = run_quiet(root5, apply=True, channel="offline")
        after5 = read_text(sh5)
        ap, ai, asuf = split_auto_block(after5)
        snap_after = snapshot(root5)
        others_same = all(snap_before[k] == snap_after[k] for k in snap_before if k != sh5)
        # rc=1 is expected: the pre-write block was stale (diff≠∅ reported).
        # 丁 was misplaced into hubA, so it re-enters unclustered as a NEW
        # member (appended after surviving 戊, per the order-preservation rule);
        # the second run must be a fixed point (diff=∅).
        expected5 = expected_inner.replace("  - [[丁]]\n  - [[戊]]",
                                           "  - [[戊]]\n  - [[丁]]")
        rc5b, _ = run_quiet(root5)
        check("T5 --apply 只动区块内（前后缀字节不变，他文件 sha 不变，写后==规则输出且再跑 diff=∅）",
              rc5 == 1 and rc5b == 0 and bp == ap and bs == asuf and bi != ai
              and ai == expected5 and others_same)

        # T6: dry-run performs zero disk writes (whole-tree sha snapshot)
        root6 = build_vault("t6", _sh(comment, stale))
        snap6a = snapshot(root6)
        rc6, _ = run_quiet(root6)
        snap6b = snapshot(root6)
        check("T6 dry-run 零写盘（全树 sha 不变；stale 区块报 diff rc=1）",
              snap6a == snap6b and rc6 == 1)

        # T7: broken abstracts edge -> fail-loud, never silently dropped
        root7 = build_vault("t7", _sh(comment, listing),
                            extra={"02-Cards/hubA.md": _card(
                                "hubA", hub=True, abstracts=("甲", "幽灵卡"), seq=1)})
        try:
            run_quiet(root7)
            t7 = False
        except AtlasFillError as e:
            t7 = "abstracts" in str(e) and "幽灵卡" in str(e)
        check("T7 枢纽 abstracts 断链 fail-loud（点名枢纽与目标）", t7)

        # T8: CRLF fixture — regenerated block keeps CRLF, comment verbatim,
        # and dry-run diff=∅ against the CRLF manual block
        root8 = build_vault("t8", _sh(comment, listing, eol="\r\n"))
        rc8, out8 = run_quiet(root8)
        txt8 = read_text(os.path.join(root8, SH_REL.replace("/", os.sep)))
        _, inner8, _ = split_auto_block(txt8)
        cards8 = load_cards(root8)
        members8 = collect_members(cards8)
        new8, _, _ = generate_block(cards8, members8, leading_comment_lines(inner8),
                                    "\r\n", prev_inner=inner8)
        check("T8 CRLF 保持 + 注释行原样保留（不硬编码规则文本）",
              rc8 == 0 and "diff=∅" in out8 and new8 == inner8 and "\r\n" in new8
              and "\n" not in new8.replace("\r\n", "")
              and new8.startswith("\r\n" + comment[0] + "\r\n"))

        # T9: over soft budget -> loud report (rc=1), no truncation
        many = {}
        big_listing = list(listing)
        for i in range(60):
            nm = "预算填充卡名称加长第%02d号" % i
            many["02-Cards/%s.md" % nm] = _card(nm, seq=100 + i)
            big_listing.append("  - [[%s]]" % nm)
        root9 = build_vault("t9", _sh(comment, big_listing), extra=many)
        rc9, out9 = run_quiet(root9)
        check("T9 超软预算响亮报告（rc=1，输出含超预算，不截断）",
              rc9 == 1 and "超预算" in out9)

        # T10: SH missing -> fail-loud (named error, non-zero via main)
        root10 = build_vault("t10", _sh(comment, listing))
        os.unlink(os.path.join(root10, SH_REL.replace("/", os.sep)))
        try:
            run_quiet(root10)
            t10 = False
        except AtlasFillError as e:
            t10 = "SH 缺失" in str(e)
        check("T10 SH 缺失 fail-loud（AtlasFillError，非静默）", t10)
    finally:
        shutil.rmtree(tmp, ignore_errors=True)

    ok = all(v for _, v in results)
    print("\natlas_fill --self-test: %s（%d 项）" % ("PASS" if ok else "FAIL", len(results)))
    return 0 if ok else 1


def main():
    ap = argparse.ArgumentParser(description="SH auto 区块投影生成器（刀G）")
    ap.add_argument("--vault", help="vault 根（默认 ttl_locator 解析）")
    ap.add_argument("--channel", default="cli", choices=["cli", "offline"],
                    help="写入通道（默认官方 CLI，不可用显式降级 offline）")
    ap.add_argument("--apply", action="store_true", help="实际写入（默认 dry-run）")
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
    try:
        return run(loc.root, apply=a.apply, channel=a.channel)
    except AtlasFillError as exc:
        print("ERROR: %s" % exc)
        return 2


if __name__ == "__main__":
    try:
        sys.exit(main())
    except Exception as exc:
        print("atlas_fill 自身失败: %r" % exc)
        sys.exit(2)
````
