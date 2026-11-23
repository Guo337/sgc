---
id: vol-vol-extract
type: script-volume
created: 2026-11-16
last-touched: 2026-11-16
source-of: 90-Meta/scripts/vol_extract.py
fence: "````"
eol: crlf
sha256: 79154cef8130ddeb7f03b05c23dda22ddecb120b68a91c1baed338937e0cc4c3
---
````python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
"""vol_extract.py — script-volume extractor (stdlib only).

Extracts .py runtime files from script volumes (.md with a ````python code block).
Resolves inter-volume dependencies declared in frontmatter `depends-on` (vol ids),
topologically sorts them, and extracts in dependency order.

Usage:
  python vol_extract.py <vol-id-or-path> [--vault PATH] [--out-dir PATH] [--all]
  python vol_extract.py --self-test

Examples:
  python vol_extract.py vol-entry-doctor --vault agents_vault
  python vol_extract.py vol-read-orchestrator --vault agents_vault  # recursively extracts deps
  python vol_extract.py --all --vault agents_vault                  # extract every volume

Exit codes: 0 = success / 1 = extraction error / 2 = tool self-failure.
"""
import sys, os, re, hashlib, argparse

try:
    sys.stdout.reconfigure(encoding="utf-8")
except Exception:
    pass

VERSION = "0.1.0"

OF = b"````python\r\n"
CF = b"\r\n````\r\n"
FM_RE = re.compile(rb"^---\r?\n(.*?)\r?\n---\r?\n", re.S)
VOL_TYPE_RE = re.compile(rb"(?m)^type:\s*script-volume\s*$")
ID_RE = re.compile(rb"(?m)^id:\s*(\S+)")
SHA_RE = re.compile(rb"(?m)^sha256:\s*([0-9a-f]{64})")
SOURCE_RE = re.compile(rb"(?m)^source-of:\s*(\S+)")
EOL_RE = re.compile(rb"(?m)^eol:\s*(\S+)")
DEPS_RE = re.compile(rb"(?m)^depends-on:\s*\[([^\]]*)\]")


def parse_volume(data):
    """Parse a volume file (bytes). Returns dict with id, sha256, source_of, eol, deps, code.
    Raises ValueError on structural problems."""
    m = FM_RE.match(data)
    if not m:
        raise ValueError("no frontmatter")
    fm = m.group(1)
    if not VOL_TYPE_RE.search(fm):
        raise ValueError("not type: script-volume")
    mid = ID_RE.search(fm)
    if not mid:
        raise ValueError("no id in frontmatter")
    vol_id = mid.group(1).decode().strip("\"'")
    msha = SHA_RE.search(fm)
    declared_sha = msha.group(1).decode() if msha else None
    msrc = SOURCE_RE.search(fm)
    source_of = msrc.group(1).decode().strip("\"'") if msrc else None
    meol = EOL_RE.search(fm)
    eol = meol.group(1).decode().strip("\"'") if meol else "crlf"
    mdeps = DEPS_RE.search(fm)
    deps = []
    if mdeps:
        deps = [x.strip().strip("\"'") for x in mdeps.group(1).decode().split(",") if x.strip()]
    # extract code block: open fence first occurrence, close fence LAST occurrence
    s = data.find(OF)
    if s < 0:
        raise ValueError("no ````python opening fence")
    s += len(OF)
    e = data.rfind(CF)
    if e < 0 or e < s:
        raise ValueError("no closing fence after code")
    code = data[s:e]
    # verify sha256
    if declared_sha:
        actual = hashlib.sha256(code).hexdigest()
        if actual != declared_sha:
            raise ValueError("sha256 mismatch: declared %s… actual %s…" % (declared_sha[:12], actual[:12]))
    return {"id": vol_id, "sha256": declared_sha, "source_of": source_of,
            "eol": eol, "deps": deps, "code": code}


def scan_volumes(vault_root):
    """Walk vault, find all script-volume .md files. Returns {vol_id: (path, parsed)}."""
    volumes = {}
    for dp, dn, fn in os.walk(vault_root):
        dn[:] = [d for d in dn if d not in (".obsidian", ".git", "node_modules", "__pycache__", ".trash")]
        for f in fn:
            if not f.endswith(".md"):
                continue
            full = os.path.join(dp, f)
            data = open(full, "rb").read()
            if b"type: script-volume" not in data[:500]:
                continue
            try:
                parsed = parse_volume(data)
            except ValueError:
                continue
            volumes[parsed["id"]] = (full, parsed)
    return volumes


def topo_sort(vol_id, volumes, visiting=None, done=None):
    """DFS topological sort of dependency graph. Returns list of vol_ids in extraction order.
    Raises ValueError on missing dep or cycle."""
    if visiting is None:
        visiting = set()
    if done is None:
        done = set()
    if vol_id in done:
        return []
    if vol_id in visiting:
        raise ValueError("circular dependency at %s" % vol_id)
    if vol_id not in volumes:
        raise ValueError("dependency %s not found in vault" % vol_id)
    visiting.add(vol_id)
    order = []
    for dep in volumes[vol_id][1]["deps"]:
        order.extend(topo_sort(dep, volumes, visiting, done))
    visiting.discard(vol_id)
    done.add(vol_id)
    order.append(vol_id)
    return order


def extract_one(vol_id, volumes, out_dir):
    """Extract a single volume's .py to out_dir/source-of basename. Returns output path."""
    path, parsed = volumes[vol_id]
    code = parsed["code"]
    source_of = parsed["source_of"]
    if not source_of:
        raise ValueError("%s: no source-of declared" % vol_id)
    basename = os.path.basename(source_of)
    out_path = os.path.join(out_dir, basename)
    os.makedirs(out_dir, exist_ok=True)
    open(out_path, "wb").write(code)
    return out_path


def self_test():
    """In-memory fixtures: verify parse, topo sort, cycle detection, sha mismatch."""
    ok = True
    # well-formed volume
    code = b"print('hello')"
    sha = hashlib.sha256(code).hexdigest()
    vol = (b"---\r\nid: vol-test\r\ntype: script-volume\r\ncreated: 2026-11-16\r\n"
           b"last-touched: 2026-11-16\r\nsource-of: 90-Meta/scripts/test.py\r\n"
           b'fence: "````"\r\neol: crlf\r\nsha256: ' + sha.encode() + b"\r\n---\r\n"
           + OF + code + CF)
    p = parse_volume(vol)
    t1 = p["id"] == "vol-test" and p["code"] == code and p["deps"] == []
    ok = ok and t1
    print("  [%s] parse clean volume" % ("OK" if t1 else "FAIL"))
    # deps parsing
    vol2 = vol.replace(b"sha256:", b"depends-on: [vol-dep-a, vol-dep-b]\r\nsha256:")
    p2 = parse_volume(vol2)
    t2 = p2["deps"] == ["vol-dep-a", "vol-dep-b"]
    ok = ok and t2
    print("  [%s] parse depends-on" % ("OK" if t2 else "FAIL"))
    # sha mismatch
    bad = vol.replace(sha.encode(), b"0" * 64)
    try:
        parse_volume(bad)
        t3 = False
    except ValueError as e:
        t3 = "sha256 mismatch" in str(e)
    ok = ok and t3
    print("  [%s] sha256 mismatch detected" % ("OK" if t3 else "FAIL"))
    # topo sort: a depends on b, b depends on c -> order [c, b, a]
    vols = {
        "vol-a": ("", {"deps": ["vol-b"], "code": b"", "source_of": "a.py"}),
        "vol-b": ("", {"deps": ["vol-c"], "code": b"", "source_of": "b.py"}),
        "vol-c": ("", {"deps": [], "code": b"", "source_of": "c.py"}),
    }
    order = topo_sort("vol-a", vols)
    t4 = order == ["vol-c", "vol-b", "vol-a"]
    ok = ok and t4
    print("  [%s] topo sort dependency order" % ("OK" if t4 else "FAIL"))
    # cycle detection
    vols["vol-c"][1]["deps"] = ["vol-a"]
    try:
        topo_sort("vol-a", vols)
        t5 = False
    except ValueError as e:
        t5 = "circular" in str(e)
    ok = ok and t5
    print("  [%s] circular dependency detected" % ("OK" if t5 else "FAIL"))
    # missing dep
    vols2 = {"vol-x": ("", {"deps": ["vol-nonexist"], "code": b"", "source_of": "x.py"})}
    try:
        topo_sort("vol-x", vols2)
        t6 = False
    except ValueError as e:
        t6 = "not found" in str(e)
    ok = ok and t6
    print("  [%s] missing dependency detected" % ("OK" if t6 else "FAIL"))
    print("\nvol_extract --self-test: %s" % ("PASS" if ok else "FAIL"))
    return 0 if ok else 1


def main():
    ap = argparse.ArgumentParser(description="script-volume extractor")
    ap.add_argument("target", nargs="?", help="vol-id or .md path")
    ap.add_argument("--vault", help="vault root (default: auto-detect via ttl_locator)")
    ap.add_argument("--out-dir", help="output directory for .py (default: <vault>/90-Meta/scripts)")
    ap.add_argument("--all", action="store_true", help="extract all volumes")
    ap.add_argument("--self-test", action="store_true")
    a = ap.parse_args()

    if a.self_test:
        return self_test()

    # resolve vault root
    vault = a.vault
    if not vault:
        try:
            from ttl_locator import resolve_vault   # optional dep: vol-ttl-locator (fallback below)
            vault = resolve_vault(None).root
        except Exception:
            vault = os.path.abspath(os.path.join(os.path.dirname(__file__), "..", ".."))
    if not os.path.isdir(vault):
        print("vault not found: %s" % vault)
        return 2

    volumes = scan_volumes(vault)
    if not volumes:
        print("no script-volumes found in %s" % vault)
        return 1

    out_dir = a.out_dir or os.path.join(vault, "90-Meta", "scripts")

    if a.all:
        targets = list(volumes.keys())
    elif a.target:
        # accept vol-id or file path
        if a.target in volumes:
            targets = [a.target]
        elif os.path.isfile(a.target):
            data = open(a.target, "rb").read()
            p = parse_volume(data)
            targets = [p["id"]]
            volumes[p["id"]] = (a.target, p)
        else:
            print("target not found: %s (known ids: %s)" % (a.target, ", ".join(sorted(volumes))))
            return 1
    else:
        ap.print_help()
        return 2

    # resolve full dependency closure and extract in order
    extracted = []
    try:
        for t in targets:
            order = topo_sort(t, volumes)
            for vol_id in order:
                if vol_id in extracted:
                    continue
                out_path = extract_one(vol_id, volumes, out_dir)
                extracted.append(vol_id)
                print("extracted %s -> %s" % (vol_id, out_path))
    except ValueError as e:
        print("ERROR: %s" % e)
        return 1
    print("done: %d volume(s) extracted" % len(extracted))
    return 0


if __name__ == "__main__":
    try:
        sys.exit(main())
    except Exception as e:
        print("vol_extract self-failure: %r" % e)
        sys.exit(2)
````
