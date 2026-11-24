---
id: vol-fed-sync
type: script-volume
created: 2026-11-25
last-touched: 2026-11-25
source-of: 90-Meta/scripts/fed_sync.py
fence: "````"
eol: crlf
depends-on: []
sha256: be9cee4e9edbf80a5a07e4defa0062c055273d163f3da20ec2d20800253b94d6
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

VERSION = "0.1.1"
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
            if i1 == i2:
                continue  # zero-width insert handled at boundary points
            if i2 <= a or i1 >= b:
                continue
            if tag == "equal":
                out.extend(side_lines[j1:j2])
            else:
                changed = True
                out.extend(side_lines[j1:j2])
        return out if changed else None

    def ins_at(ops, p, side_lines):
        """Pure insertion at base position p (zero-width op), or None.
        v0.1.1 fix: region walk over (a,b) pairs skipped zero-width ops,
        silently dropping insertions (local deviation loss = contract breach)."""
        for tag, i1, i2, j1, j2 in ops:
            if i1 == i2 == p and tag == "insert":
                return side_lines[j1:j2]
        return None

    for idx, a in enumerate(boundaries):
        # insertions anchored at this boundary point (before base[a:])
        li = ins_at(l_ops, a, local)
        ui = ins_at(u_ops, a, upstream)
        if li is not None and ui is not None:
            if li == ui:
                merged.extend(li)
            else:
                conflicts += 1
                merged.append("<<<<<<< LOCAL")
                merged.extend(li)
                merged.append("=======")
                merged.extend(ui)
                merged.append(">>>>>>> UPSTREAM")
        elif li is not None:
            merged.extend(li)
        elif ui is not None:
            merged.extend(ui)
        if idx + 1 >= len(boundaries):
            break
        b = boundaries[idx + 1]
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


def _self_test():
    """Regression suite for the merge fact feeder (v0.1.1 insertion fix)."""
    def mg(b, l, u):
        m, c = _rcs_merge(b.splitlines(), l.splitlines(), u.splitlines())
        return "\n".join(m), c

    cases = []
    # 1 disjoint insertions both kept (was: silently dropped, v0.1.0 bug)
    out, c = mg("A\nB\nC", "A\nX\nB\nC", "A\nB\nY\nC")
    cases.append(("disjoint-insert", c == 0 and out == "A\nX\nB\nY\nC"))
    # 2 delete + append
    out, c = mg("A\nB\nC\nD", "A\nC\nD", "A\nB\nC\nZ")
    cases.append(("delete-append", c == 0 and out == "A\nC\nZ"))
    # 3 identical change both sides
    out, c = mg("A\nB\nC", "A\nB2\nC", "A\nB2\nC")
    cases.append(("same-change", c == 0 and out == "A\nB2\nC"))
    # 4 head insertion local-only (was: dropped)
    out, c = mg("A\nB", "X\nA\nB", "A\nB")
    cases.append(("head-insert", c == 0 and out == "X\nA\nB"))
    # 5 same-position different insertions = true conflict
    out, c = mg("A\nB", "A\nX\nB", "A\nY\nB")
    cases.append(("same-pos-conflict", c == 1 and "<<<<<<< LOCAL" in out
                  and "X" in out and "Y" in out))
    # 6 tail insertion upstream-only (was: dropped)
    out, c = mg("A\nB", "A\nB", "A\nB\nZ")
    cases.append(("tail-insert", c == 0 and out == "A\nB\nZ"))
    # 7 unchanged both sides
    out, c = mg("A\nB", "A\nB", "A\nB")
    cases.append(("no-change", c == 0 and out == "A\nB"))
    # 8 same-position identical insertions dedup
    out, c = mg("A\nB", "A\nX\nB", "A\nX\nB")
    cases.append(("same-insert-dedup", c == 0 and out == "A\nX\nB"))
    # 9 sections report
    secs = _sections("## S1\nx\n## S2\ny")
    cases.append(("sections-parse", set(secs) == {"S1", "S2"}))
    bad = [n for n, ok in cases if not ok]
    if bad:
        print("fed_sync --self-test: FAIL " + ", ".join(bad))
        return 1
    print("fed_sync --self-test: PASS (%d cases)" % len(cases))
    return 0


def main(argv=None):
    ap = argparse.ArgumentParser(description="federation spec re-sync three-way merge (offline, stdlib)")
    ap.add_argument("--version", action="version", version=VERSION)
    ap.add_argument("--self-test", action="store_true", help="run regression suite")
    sub = ap.add_subparsers(dest="cmd", required=False)
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
    if args.self_test:
        return _self_test()
    if not args.cmd:
        ap.error("a subcommand is required (merge/sections) or --self-test")
    return args.fn(args)


if __name__ == "__main__":
    raise SystemExit(main())
````
