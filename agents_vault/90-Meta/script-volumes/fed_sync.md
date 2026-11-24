---
id: vol-fed-sync
type: script-volume
created: 2026-11-25
last-touched: 2026-11-25
source-of: 90-Meta/scripts/fed_sync.py
fence: "````"
eol: crlf
depends-on: []
sha256: fb10afe75c1e769cc8ad7a2f7abd63e74e82a3eb29b4738f3d52c6441f71e5ad
---
````python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
"""fed_sync.py -- offline three-way merge helper for federation spec re-sync.

Revision-rights contract (corr-112/117/119) upgrade leg: given a copied spec's
upstream baseline (cache), the local current text, and a new upstream version,
produce a three-way merged result. Conflicts are marked for human/Agent ruling,
never auto-resolved (conffile/ucf/git merge-file isomorph; stdlib difflib only,
no git dependency, offline).

Design boundary (Owner philosophy: strong text constraint, weak script
constraint): this is a merge FACT feeder, not an adjudication engine. It marks
conflicts and reports section-set changes; semantic ruling belongs to Owner.

Subcommands:
  merge --base <file> --local <file> --upstream <file> [--out <file>]
      Three-way merge. Exit 0 = clean, 1 = conflicts marked, 2 = usage error.
  sections --base <file> --upstream <file>
      Report H2 section-set changes (added/removed/renamed-candidate) for ruling.

Pure stdlib. English comments by project convention; user-facing text Chinese.
"""
from __future__ import annotations

import argparse
import difflib
import io
import re
import sys

VERSION = "0.1.0"
H2_RE = re.compile(r"(?m)^##\s+(.+?)\s*$")


def _read(path):
    with io.open(path, "r", encoding="utf-8", newline="") as f:
        return f.read().replace("\r\n", "\n")


def _sections(text):
    secs, cur, buf = {}, None, []
    for ln in text.splitlines():
        m = H2_RE.match(ln)
        if m:
            if cur is not None:
                secs[cur] = "\n".join(buf)
            cur, buf = m.group(1).strip(), []
        elif cur is not None:
            buf.append(ln)
    if cur is not None:
        secs[cur] = "\n".join(buf)
    return secs


def _rcs_merge(base, local, upstream):
    """Three-way merge by aligning local and upstream against base with
    SequenceMatcher, walking base regions. Conflicts marked, never auto-ruled."""
    merged, conflicts = [], 0
    lm = difflib.SequenceMatcher(None, base, local, autojunk=False)
    um = difflib.SequenceMatcher(None, base, upstream, autojunk=False)
    l_ops = lm.get_opcodes()
    u_ops = um.get_opcodes()
    n = len(base)
    # Region walk: process base in chunks delimited by opcode boundaries.
    boundaries = sorted({0, n}
                        | {op[1] for op in l_ops} | {op[2] for op in l_ops}
                        | {op[1] for op in u_ops} | {op[2] for op in u_ops})
    boundaries = [b for b in boundaries if 0 <= b <= n]

    def repl_for(ops, a, b, side_lines):
        """Replacement lines for base[a:b] on one side, or None if unchanged."""
        out = []
        changed = False
        for tag, i1, i2, j1, j2 in ops:
            if i2 <= a or i1 >= b:
                continue
            if tag == "equal":
                out.extend(side_lines[j1:j2])
            else:
                changed = True
                out.extend(side_lines[j1:j2])
        return out if changed else None

    for a, b in zip(boundaries, boundaries[1:]):
        if a == b:
            continue
        lr = repl_for(l_ops, a, b, local)
        ur = repl_for(u_ops, a, b, upstream)
        if lr is None and ur is None:
            merged.extend(base[a:b])
        elif lr is None:
            merged.extend(ur)
        elif ur is None:
            merged.extend(lr)
        elif lr == ur:
            merged.extend(lr)
        else:
            conflicts += 1
            merged.append("<<<<<<< LOCAL")
            merged.extend(lr)
            merged.append("=======")
            merged.extend(ur)
            merged.append(">>>>>>> UPSTREAM")
    return merged, conflicts


def cmd_merge(args):
    base = _read(args.base).splitlines()
    local = _read(args.local).splitlines()
    upstream = _read(args.upstream).splitlines()
    merged, conflicts = _rcs_merge(base, local, upstream)
    text = "\n".join(merged) + "\n"
    if args.out:
        with io.open(args.out, "w", encoding="utf-8", newline="") as f:
            f.write(text)
    else:
        sys.stdout.write(text)
    if conflicts:
        sys.stderr.write("[fed_sync] %d 处冲突已落标记，须人/Agent 裁决（不自动裁语义冲突）\n" % conflicts)
        return 1
    sys.stderr.write("[fed_sync] 三方合并干净，无冲突\n")
    return 0


def cmd_sections(args):
    base = set(_sections(_read(args.base)))
    up = set(_sections(_read(args.upstream)))
    added = sorted(up - base)
    removed = sorted(base - up)
    print("## 节集合变化（须显式呈裁，不静默模糊匹配）")
    print("新增节（默认红线管辖）：", added or "无")
    print("删除节（须确认本地声明处置）：", removed or "无")
    return 0


def main(argv=None):
    ap = argparse.ArgumentParser(description="federation spec re-sync three-way merge (offline, stdlib)")
    ap.add_argument("--version", action="version", version=VERSION)
    sub = ap.add_subparsers(dest="cmd", required=True)
    pm = sub.add_parser("merge", help="three-way merge base/local/upstream")
    pm.add_argument("--base", required=True)
    pm.add_argument("--local", required=True)
    pm.add_argument("--upstream", required=True)
    pm.add_argument("--out")
    pm.set_defaults(fn=cmd_merge)
    ps = sub.add_parser("sections", help="report H2 section-set changes")
    ps.add_argument("--base", required=True)
    ps.add_argument("--upstream", required=True)
    ps.set_defaults(fn=cmd_sections)
    args = ap.parse_args(argv)
    return args.fn(args)


if __name__ == "__main__":
    raise SystemExit(main())
````
