---
id: vol-read-audit
type: script-volume
created: 2026-11-16
last-touched: 2026-11-16
source-of: 90-Meta/scripts/read_audit.py
fence: "````"
eol: crlf
sha256: a58acbb97922c12aa7d528aaab515d1439ff1773abf4dc880b57758fa557b4b2
---
````python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
"""read_audit.py — read-channel compliance audit (stdlib only, read-only).

Audits whether vault-internal reads went through the Obsidian link channel
(MCP / REST / official CLI) vs raw disk reads (view/grep/glob). SPEC-READ
requires the Obsidian channel as the primary read path so that SH routing,
links and backlinks are actually used; raw disk reads bypass the link graph.

This is a POST-HOC audit (medium enforcement): it cannot intercept tool calls
at the harness layer, but it makes disk-read drift discoverable and reportable.
Reads the harness session event log (events.jsonl), never the vault content.

Usage:
  python read_audit.py --events <events.jsonl> [--vault-name agents_vault]
                       [--threshold 0.2] [--json]

Exit codes: 0 = within threshold / 1 = disk-read ratio exceeds threshold /
            2 = tool self-failure (missing log, parse error).
"""
import argparse
import json
import os
import sys

VERSION = "0.1.0"

DISK_TOOLS = {"view", "grep", "glob", "rg"}
OBSIDIAN_PREFIX = "obsidian-"


def audit(events_path, vault_name, threshold):
    if not os.path.isfile(events_path):
        print("ERROR: events log not found: %s" % events_path)
        return 2
    disk = 0
    mcp = 0
    first_disk = None
    samples = []
    with open(events_path, encoding="utf-8") as handle:
        for line in handle:
            line = line.strip()
            if not line:
                continue
            try:
                ev = json.loads(line)
            except ValueError:
                continue
            if ev.get("type") != "tool.execution_start":
                continue
            data = ev.get("data", {})
            tool = data.get("toolName", "")
            args = json.dumps(data.get("arguments", {}), ensure_ascii=False)
            if vault_name not in args:
                continue  # only count vault-internal reads
            if tool in DISK_TOOLS:
                disk += 1
                if first_disk is None:
                    first_disk = ev.get("timestamp")
                if len(samples) < 5:
                    samples.append("%s %s" % (ev.get("timestamp"), tool))
            elif tool.startswith(OBSIDIAN_PREFIX):
                mcp += 1
    total = disk + mcp
    ratio = (disk / total) if total else 0.0
    verdict = "PASS" if ratio <= threshold else "DRIFT"
    result = {
        "tool": "read_audit",
        "version": VERSION,
        "vault_name": vault_name,
        "vault_internal_disk_reads": disk,
        "vault_internal_obsidian_reads": mcp,
        "total_vault_reads": total,
        "disk_ratio": round(ratio, 4),
        "threshold": threshold,
        "verdict": verdict,
        "first_disk_read": first_disk,
        "disk_samples": samples,
    }
    return result


def main():
    ap = argparse.ArgumentParser(description="read-channel compliance audit")
    ap.add_argument("--events", required=True, help="path to events.jsonl")
    ap.add_argument("--vault-name", default="agents_vault")
    ap.add_argument("--threshold", type=float, default=0.2,
                    help="max allowed disk-read ratio (default 0.2)")
    ap.add_argument("--json", action="store_true")
    args = ap.parse_args()
    result = audit(args.events, args.vault_name, args.threshold)
    if isinstance(result, int):
        return result
    if args.json:
        print(json.dumps(result, ensure_ascii=False, indent=2))
    else:
        print("read_audit v%s — vault=%s" % (VERSION, result["vault_name"]))
        print("  库内磁盘读: %d | Obsidian 通道读: %d | 合计: %d"
              % (result["vault_internal_disk_reads"],
                 result["vault_internal_obsidian_reads"],
                 result["total_vault_reads"]))
        print("  磁盘读占比: %.1f%% (阈值 %.0f%%)"
              % (result["disk_ratio"] * 100, result["threshold"] * 100))
        print("  判定: %s" % result["verdict"])
        if result["first_disk_read"]:
            print("  首次磁盘读: %s" % result["first_disk_read"])
        if result["verdict"] == "DRIFT":
            print("  ⚠ 读取通道漂移：库内读取应以 Obsidian 链接通道为主（SPEC-READ），"
                  "磁盘读仅兜底。磁盘读绕开 SH 路由/链接/反链。")
    return 0 if result["verdict"] == "PASS" else 1


if __name__ == "__main__":
    sys.exit(main())
````
