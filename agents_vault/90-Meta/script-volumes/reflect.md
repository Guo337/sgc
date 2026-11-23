---
id: vol-reflect
type: script-volume
created: 2026-11-16
last-touched: 2026-11-20
source-of: 90-Meta/scripts/reflect.py
fence: "````"
eol: crlf
depends-on: [vol-ttl-locator]
sha256: 637af17efc42c6ead7666104177dcb50fa928f1ea9b33873b363451fa2939e58
---
````python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
"""
reflect.py — reflection compiler (minimal version; semi-automatic rollup before Stage 2)

Role (SPEC-WRITE "fully-automatic archiving" section):
  Roll up the "## 候选结晶" (candidate-crystallization) section of the logs ->
  dedupe / classify -> draft L2 seed cards; on apply, backfill the source logs'
  reflect-to derived pointers.

Write permissions (SPEC-WRITE permission matrix, reflect.py --apply row):
  allow 04-Logs : backfill reflect-to (frontmatter derived pointer, not body tampering)
  allow 02-Cards: generate seed cards (provenance: reflect, maturity: seed, cite log id)
  deny  03-Atlas / 01-Specs / 90-Meta
    -> L3 pitfall candidates (01-Specs) are NOT written by this script; they are listed
       in a "manual route" worklist for the Agent (micro-feedback flow) or a human.
       Atlas reachability registration (auto block) is likewise maintained manually.

Usage:
  python 90-Meta/scripts/reflect.py                    # dry-run: numbered worklist (read-only)
  python 90-Meta/scripts/reflect.py --json             # machine-readable worklist
  python 90-Meta/scripts/reflect.py --apply 1,3        # write chosen L2 cards + backfill reflect-to
  python 90-Meta/scripts/reflect.py --apply 1 --force  # override the already-crystallized guard
  python 90-Meta/scripts/reflect.py --self-test        # built-in fixture self-test

Design: read-only by default, explicit --apply (SPEC-WRITE safety B); stdlib only;
does not re-implement the link engine.
NOTE: comments/docstrings are English by project convention; user-facing report text
and generated vault content stay Chinese (the vault is a Chinese library).
"""

import argparse
import datetime
import json
import os
import re
import sys
from ttl_locator import resolve_vault   # depends-on: vol-ttl-locator

try:
    sys.stdout.reconfigure(encoding="utf-8")
except Exception:
    pass

VERSION = "0.3.1"

LOG_DIR = "04-Logs/logs"          # nested split: session logs live in 04-Logs/logs/
LEGACY_LOG_DIR = "04-Logs"        # pre-split flat layout (fallback during migration)
CARD_DIR = "02-Cards"
SECTION = "## 候选结晶"

# A candidate is treated as already done when any of these appears anywhere in the bullet.
DONE_RE = re.compile(r"(已直接建卡|已建卡|已结晶|已直接结晶|已完成)")

# Log id pattern, used to recover which logs a reflect card was compiled from.
LOG_ID_RE = re.compile(r"log-\d{4}-\d{2}-\d{2}-\d+")

# Card-type keywords -> (layer, type). Pitfall tokens map to L3 (manual route, not written here).
TYPE_HINTS = [
    (re.compile(r"pitfall|⚠️|坑", re.I), (3, "pitfall")),
    (re.compile(r"procedure|流程|速查|清单|纪律|方法|步骤", re.I), (2, "procedure")),
    (re.compile(r"observation|佐证|实测|观察", re.I), (2, "observation")),
    (re.compile(r"concept|原则|原理|准则|模型|概念", re.I), (2, "concept")),
    (re.compile(r"claim|断言|主张", re.I), (2, "claim")),
]

# Explicit card-type declarations take priority over the keyword heuristics above,
# so a bullet that merely mentions a pitfall token but declares "-> procedure card" is not
# misrouted to L3.
DECL_ARROW_RE = re.compile(
    r"(?:→|->|结晶为)\s*(pitfall|procedure|concept|observation|claim|entity)", re.I)
DECL_CARD_RE = re.compile(
    r"(pitfall|procedure|concept|observation|claim|entity)\s*卡", re.I)


# ---------------------------------------------------------------- basic parsing

def vault_root(cli):
    here = os.path.abspath(os.path.join(os.path.dirname(__file__), "..", ".."))
    return resolve_vault(cli, script_fallback=here).root


def parse_fm(text):
    """Minimal frontmatter parser: return (dict|None, body). Only top-level `key: value`.

    Files are read with universal newlines, so CRLF on disk becomes LF in memory;
    `find("\\n---")` therefore works regardless of the on-disk line ending."""
    if not text.startswith("---"):
        return None, text
    end = text.find("\n---", 3)
    if end < 0:
        return None, text
    raw = text[3:end].strip("\n")
    body = text[end + 4:].lstrip("\n")
    fm = {}
    for line in raw.splitlines():
        m = re.match(r"^([A-Za-z0-9_\-]+):\s*(.*)$", line)
        if m:
            fm[m.group(1)] = m.group(2).strip()
    return fm, body


def fm_get_id(fm):
    if not fm:
        return None
    v = fm.get("id", "")
    return v.strip().strip('"').strip("'") or None


def fm_get_list(fm, key):
    """Loosely parse `key: [a, b]` or `key: ["a","b"]` into list[str]."""
    if not fm:
        return []
    v = (fm.get(key, "") or "").strip()
    if not v or v == "[]":
        return []
    if v.startswith("[") and v.endswith("]"):
        inner = v[1:-1]
        return [x.strip().strip('"').strip("'") for x in inner.split(",") if x.strip()]
    return [v.strip().strip('"').strip("'")]


# ---------------------------------------------------------------- candidate extraction

def extract_candidates(body):
    """Return the bullet lines of the '## 候选结晶' section (up to the next '## ' or EOF)."""
    idx = body.find(SECTION)
    if idx < 0:
        return []
    rest = body[idx + len(SECTION):]
    nxt = rest.find("\n## ")
    if nxt >= 0:
        rest = rest[:nxt]
    out = []
    for line in rest.splitlines():
        s = line.strip()
        if s.startswith("- ") or s.startswith("* "):
            out.append(s[2:].strip())
        elif s.startswith("-") and len(s) > 1 and s[1] != " ":
            out.append(s[1:].strip())
    return out


def strip_lead(bullet):
    """Drop the '待结晶：' / '待结晶:' lead-in; return the core description."""
    s = bullet.strip()
    s = re.sub(r"^待结晶\s*[:：]\s*", "", s)
    s = re.sub(r"^待\s*结晶\s*[:：]?\s*", "", s)
    return s.strip()


def title_hint(bullet):
    """Extract a card-name hint: prefer a **bold** span, then a "quoted" span,
    then the leading phrase before the first paren/arrow/comma."""
    s = strip_lead(bullet)
    m = re.search(r"\*\*(.+?)\*\*", s)
    if m:
        t = m.group(1)
    else:
        m = re.search(r'["“](.+?)["”]', s)
        if m:
            t = m.group(1)
        else:
            t = re.split(r"[（(→]|->|，|,|\s—\s", s)[0]
    t = t.strip().strip("*").strip()
    t = t.strip('"“”\'‘’').strip()
    return t[:60]


def classify(bullet):
    """Return (layer, type). Prefer an explicit card-type declaration; fall back to
    keyword heuristics only when there is no declaration."""
    m = DECL_ARROW_RE.search(bullet) or DECL_CARD_RE.search(bullet)
    if m:
        typ = m.group(1).lower()
        return (3, "pitfall") if typ == "pitfall" else (2, typ)
    for rx, (layer, typ) in TYPE_HINTS:
        if rx.search(bullet):
            return layer, typ
    return 2, "concept"


def slugify(title):
    """Dedupe key: lowercase, strip punctuation/whitespace/decorations."""
    s = title.lower()
    s = re.sub(r"[\s\*\"'“”‘’（）()【】\[\]→↔·、,，.。:：;；/\\\-—_|+]+", "", s)
    return s


def clean_title(raw_title):
    """Strip backticks / stray quotes / trailing generic words (模式·卡) into a usable name."""
    t = raw_title.replace("`", "").strip()
    t = t.strip("\"'“”‘’").strip()
    t = re.sub(r"\s*(模式|卡)$", "", t).strip()
    return t or raw_title


# ---------------------------------------------------------------- scan / rollup

def scan_logs(root):
    """Scan session logs -> [{log_id, path, abs, reflect_to, candidates}] (sorted by name).

    Nested split: logs live in 04-Logs/logs/. Falls back to the legacy flat
    04-Logs/ root (only .md directly there) so the tool keeps working before the
    migration completes. Corrections (04-Logs/corrections/) are NOT scanned:
    they are pointer-depth footnotes, not crystallization sources (U-1).
    """
    d = os.path.join(root, LOG_DIR)
    rel_base = LOG_DIR
    if not os.path.isdir(d):
        d = os.path.join(root, LEGACY_LOG_DIR)
        rel_base = LEGACY_LOG_DIR
    logs = []
    if not os.path.isdir(d):
        return logs
    for fn in sorted(os.listdir(d)):
        if not fn.endswith(".md"):
            continue
        full = os.path.join(d, fn)
        if not os.path.isfile(full):
            continue
        with open(full, encoding="utf-8") as fh:
            txt = fh.read()
        fm, body = parse_fm(txt)
        logs.append({
            "log_id": fm_get_id(fm) or fn[:-3],
            "path": os.path.join(rel_base, fn).replace("\\", "/"),
            "abs": full,
            "reflect_to": fm_get_list(fm, "reflect-to"),
            "candidates": extract_candidates(body),
        })
    return logs


def existing_card_titles(root):
    """Return (titles, ids) of existing 02-Cards (title = filename without .md).

    Recursive: nested subdirectories (e.g. 02-Cards/incubator/) are part of the
    card namespace too; a flat listdir would miss incubated cards and reopen a
    duplicate-crystallization blind spot (G1-3)."""
    d = os.path.join(root, CARD_DIR)
    titles, ids = set(), set()
    if not os.path.isdir(d):
        return titles, ids
    for dp, _dn, fns in os.walk(d):
        for fn in sorted(fns):
            if fn.endswith(".md"):
                titles.add(fn[:-3])
                with open(os.path.join(dp, fn), encoding="utf-8") as fh:
                    fm, _ = parse_fm(fh.read())
                cid = fm_get_id(fm)
                if cid:
                    ids.add(cid)
    return titles, ids


def reflect_card_sources(root):
    """Scan 02-Cards; for provenance:reflect cards, collect the source log ids referenced
    in the card text. Return {title: set(log_id)} (reflect cards only). Used by the
    idempotency guard to detect that a group's logs were already mined into a card.
    Recursive (incl. 02-Cards/incubator/): incubated reflect cards still count as
    already-mined sources (G1-3)."""
    d = os.path.join(root, CARD_DIR)
    out = {}
    if not os.path.isdir(d):
        return out
    for dp, _dn, fns in os.walk(d):
        for fn in sorted(fns):
            if not fn.endswith(".md"):
                continue
            with open(os.path.join(dp, fn), encoding="utf-8") as fh:
                txt = fh.read()
            fm, _ = parse_fm(txt)
            prov = ((fm or {}).get("provenance", "") or "").strip().strip('"').strip("'").lower()
            if prov != "reflect":
                continue
            out[fn[:-3]] = set(LOG_ID_RE.findall(txt))
    return out


def next_seq(ids):
    """Next sequence number derived from the existing c-{yyyy}-{seq} id set."""
    mx = 0
    for i in ids:
        m = re.match(r"c-\d{4}-(\d+)$", i or "")
        if m:
            mx = max(mx, int(m.group(1)))
    return mx + 1


def _ngrams(s, n=3):
    return {s[i:i + n] for i in range(len(s) - n + 1)}


def titles_similar(t1, t2):
    """Heuristic: do two titles denote the same card? True when one normalized slug
    contains the other, or they share a latin token (>=4 chars), or share a 3-gram
    (covers CJK, where there are no word separators)."""
    s1, s2 = slugify(clean_title(t1)), slugify(clean_title(t2))
    if not s1 or not s2:
        return False
    if s1 in s2 or s2 in s1:
        return True
    tok1 = set(re.findall(r"[a-z0-9_]{4,}", s1))
    tok2 = set(re.findall(r"[a-z0-9_]{4,}", s2))
    if tok1 & tok2:
        return True
    return bool(_ngrams(s1) & _ngrams(s2))


def source_covering_cards(group, card_srcs):
    """Return every existing reflect-card title whose source-log set is a superset of the
    group's sources (i.e. these logs were already mined). One log can back several cards, so
    more than one card may cover a group; the caller picks the title-similar one. (Bug 4: the
    old first-match-only version mis-downgraded a shared-source duplicate to a mere warning.)"""
    gsrc = set(group["sources"])
    if not gsrc:
        return []
    return [title for title, srcs in card_srcs.items() if gsrc <= srcs]


def rollup(root):
    """Roll up every log's candidate crystallizations into a worklist.
    Return (l2_groups, l3_items, done_items).
      l2_groups: [{slug,title,type,sources,raws,crystallized_as,maybe_cover}] (deduped)
      l3_items:  [{title,type:'pitfall',source,raw}] (manual route; reflect cannot write)
      done_items:[{source,raw}] (already built / crystallized -> skipped)
    `crystallized_as` = confident duplicate (source subset AND title similar);
    `maybe_cover` = source logs already mined but title differs (warn, do not auto-skip)."""
    logs = scan_logs(root)
    titles, _ids = existing_card_titles(root)
    card_srcs = reflect_card_sources(root)
    l2, l3, done = {}, [], []
    for lg in logs:
        for raw in lg["candidates"]:
            if DONE_RE.search(raw):
                done.append({"source": lg["log_id"], "raw": raw})
                continue
            layer, typ = classify(raw)
            th = title_hint(raw)
            if layer == 3:
                l3.append({"title": th, "type": typ, "source": lg["log_id"], "raw": raw})
                continue
            # Skip when an identically-named card already exists.
            if th in titles or clean_title(th) in titles:
                done.append({"source": lg["log_id"], "raw": raw + "（已存在同名卡，跳过）"})
                continue
            key = slugify(th)
            g = l2.get(key)
            if not g:
                g = {"slug": key, "title": th, "type": typ, "sources": [], "raws": []}
                l2[key] = g
            if lg["log_id"] not in g["sources"]:
                g["sources"].append(lg["log_id"])
            g["raws"].append(raw)
            # More restatements => more mature; prefer a specific type over the default concept.
            if typ != "concept" and g["type"] == "concept":
                g["type"] = typ
    groups = sorted(l2.values(), key=lambda g: (-len(g["sources"]), g["slug"]))
    # Annotate idempotency signals against existing reflect cards. A source log can back
    # several cards, so among all covering cards pick the title-similar one as the confident
    # duplicate; a cover with no title match is only a "maybe" warning (log reused, other card).
    for g in groups:
        covers = source_covering_cards(g, card_srcs)
        similar = [c for c in covers if titles_similar(g["title"], c)]
        g["crystallized_as"] = similar[0] if similar else None
        g["maybe_cover"] = covers[0] if (covers and not similar) else None
    return groups, l3, done


# ---------------------------------------------------------------- draft generation

def build_card(group, cid, today, domain):
    """Render an L2 seed card (skeleton, provenance: reflect). The 核心 section keeps the
    raw candidate text + source-log reference; the rest are placeholders for manual polish."""
    title = clean_title(group["title"])
    src_ref = "、".join(group["sources"])
    core = "\n".join("> %s" % r for r in group["raws"])
    summary = title[:80]
    fm = [
        "---",
        'id: "%s"' % cid,
        "layer: 2",
        "type: %s" % group["type"],
        "project: memory-lib",
        "domain: [%s]" % domain,
        "maturity: seed",
        "provenance: reflect",
        "created: %s" % today,
        "last-touched: %s" % today,
        'summary: "%s"' % summary,
        "k: []",
        "v: []",
        "e:",
        '  - { target: "[[SH-knowledge-engineering]]", rel: part_of, weight: 0.6 }',
        "aliases: []",
        "---",
    ]
    body = [
        "",
        "# %s" % title,
        "",
        "## 定义",
        "<!-- 一句话自包含定义（reflect 草稿，待人工精修） -->",
        "",
        "## 核心",
        "reflect 编译自候选结晶（来源日志：%s）：" % src_ref,
        "",
        core,
        "",
        "## 存疑",
        "<!-- 不确定处如实保留，不得洗白为断言（结晶规则第4条） -->",
        "",
        "## 应用",
        "<!-- 在本库/项目中的具体用途 -->",
        "",
    ]
    return "\n".join(fm + body)


def backfill_reflect_to(log_abs, card_ref):
    """Append card_ref to the log frontmatter's reflect-to list (derived-pointer backfill,
    not body tampering). Read/write use universal newlines, so the file's CRLF stays uniform.
    Return True when the file changed."""
    with open(log_abs, encoding="utf-8") as fh:
        txt = fh.read()
    fm, _ = parse_fm(txt)
    cur = fm_get_list(fm, "reflect-to")
    if card_ref in cur:
        return False
    cur.append(card_ref)
    new_list = "[" + ", ".join('"%s"' % x for x in cur) + "]"
    # Replace only the reflect-to line; keep the rest of the file byte-for-byte.
    # `[^\r\n]*` (not `.*`) avoids consuming the CR, and [ \t]* (not \s*) avoids the newline.
    pat = re.compile(r"^(reflect-to:[ \t]*)[^\r\n]*", re.M)
    if pat.search(txt):
        txt2 = pat.sub(lambda m: m.group(1) + new_list, txt, count=1)
    else:
        # No reflect-to line: insert before the closing '---' of the frontmatter.
        end = txt.find("\n---", 3)
        txt2 = txt[:end] + "\nreflect-to: %s" % new_list + txt[end:]
    if txt2 != txt:
        with open(log_abs, "w", encoding="utf-8") as fh:
            fh.write(txt2)
        return True
    return False


# ---------------------------------------------------------------- output

def print_worklist(groups, l3, done, root):
    print("reflect.py v%s — 候选结晶 rollup（dry-run，未落盘）" % VERSION)
    print("库根: %s" % root)
    print("")
    print("== L2 可结晶（reflect --apply 落盘 02-Cards）== 共 %d 组" % len(groups))
    for i, g in enumerate(groups, 1):
        if g.get("crystallized_as"):
            flag = "  ⚠已结晶→[[%s]]（勿重选，除非 --force）" % g["crystallized_as"]
        elif g.get("maybe_cover"):
            flag = "  ⚠来源已被[[%s]]引用（可能已结晶，选取前请核对）" % g["maybe_cover"]
        else:
            flag = ""
        print("  [%d] %-26s type=%-11s 来源×%d: %s%s"
              % (i, g["title"][:26], g["type"], len(g["sources"]),
                 ", ".join(g["sources"]), flag))
    print("")
    print("== L3 pitfall 手工路由（reflect 禁写 01-Specs，走微词反馈/人工建卡）== 共 %d 条" % len(l3))
    for it in l3:
        print("  - %-30s (来源 %s)" % (it["title"][:30], it["source"]))
    print("")
    print("== 已结晶/已建卡（跳过）== 共 %d 条" % len(done))
    for it in done:
        print("  - %s: %s" % (it["source"], it["raw"][:50]))


def apply_groups(root, groups, picks, domain_map, today, force=False):
    """Write the chosen L2 groups as seed cards + backfill reflect-to.
    Confident duplicates (crystallized_as) are skipped unless --force.
    Return (written, backfilled, skipped)."""
    _titles, ids = existing_card_titles(root)
    seq = next_seq(ids)
    card_dir = os.path.join(root, CARD_DIR)
    written, backfilled, skipped = [], [], []
    log_index = {lg["log_id"]: lg for lg in scan_logs(root)}
    for n in picks:
        if n < 1 or n > len(groups):
            print("  跳过越界编号: %d" % n)
            continue
        g = groups[n - 1]
        # Idempotency guard: confident duplicate -> skip unless --force.
        if g.get("crystallized_as") and not force:
            print("  跳过疑似已结晶 [%d] %s（来源+标题均匹配 [[%s]]；确需重建加 --force）"
                  % (n, g["title"], g["crystallized_as"]))
            skipped.append((n, g["title"], g["crystallized_as"]))
            continue
        if g.get("maybe_cover") and not g.get("crystallized_as"):
            print("  提示 [%d] %s：来源日志已被 [[%s]] reflect 引用，若非重复将继续落盘"
                  % (n, g["title"], g["maybe_cover"]))
        cid = "c-%d-%05d" % (int(today[:4]), seq)
        domain = domain_map.get(g["slug"], "knowledge-engineering")
        txt = build_card(g, cid, today, domain)
        # Filename from the title (strip illegal chars / backticks / stray quotes / generic tail).
        safe = re.sub(r"[\\/:*?\"<>|↔→`'“”‘’]+", "", g["title"]).strip()
        safe = re.sub(r"\s*(模式|卡)$", "", safe).strip()[:40] or cid
        path = os.path.join(card_dir, safe + ".md")
        if os.path.exists(path):
            print("  跳过已存在文件: 02-Cards/%s.md" % safe)
            skipped.append((n, g["title"], safe))
            continue
        with open(path, "w", encoding="utf-8") as fh:
            fh.write(txt)
        written.append((cid, safe))
        seq += 1  # advance only after a successful write (no gaps on skip)
        for src in g["sources"]:
            lg = log_index.get(src)
            if lg and backfill_reflect_to(lg["abs"], cid):
                backfilled.append((src, cid))
    return written, backfilled, skipped


# ---------------------------------------------------------------- self-test

def self_test():
    """Built-in in-memory fixtures (no disk writes): verify extraction / classification /
    dedupe / title-similarity / idempotency-cover logic."""
    print("reflect.py --self-test（内置 fixture，未写盘）")
    ok = True

    def check(cond, label, detail=""):
        nonlocal ok
        ok = ok and bool(cond)
        print("  [%s] %-9s %s" % ("OK" if cond else "MISS", label, detail))

    body = ("# log\n\n## 结论\n略\n\n## 候选结晶\n\n"
            "- 已建卡：[[PIT-x]]（教训）。\n"
            '- 待结晶：**"甲模式"(细节)→ procedure 卡，可复用。\n'
            "- 待结晶：某坑 → pitfall 卡。\n"
            '- 待结晶："甲模式"补述 → procedure 卡。\n')
    cands = extract_candidates(body)
    check(len(cands) == 4, "EXTRACT", "抽出 %d 条 bullet（期望 4）" % len(cands))
    check(DONE_RE.search(cands[0]), "DONE", "识别'已建卡'跳过项")
    check(classify(cands[1]) == (2, "procedure"), "CLASSIFY", "甲模式→(2,procedure) 实得%s" % (classify(cands[1]),))
    check(classify(cands[2]) == (3, "pitfall"), "CLASSIFY", "某坑→(3,pitfall) 实得%s" % (classify(cands[2]),))
    check(classify("- 注意点路由——坑入 PIT、约定入规格 → procedure 卡。") == (2, "procedure"),
          "CLASSIFY", "含'坑'但声明 procedure→(2,procedure)")
    check("甲模式" in title_hint(cands[1]), "TITLE", "标题提示含'甲模式' 实得'%s'" % title_hint(cands[1]))
    check(slugify(title_hint(cands[1])) == slugify(title_hint(cands[3])), "DEDUPE", "两条甲模式 slug 一致")

    # titles_similar: same card phrased differently -> True; unrelated -> False.
    check(titles_similar("`entry_doctor` 的完整性检查规范", "entry_doctor校验清单与自检法"),
          "SIMILAR", "entry_doctor 变体判同")
    check(titles_similar('冷启动可发现性三路径"模式', "冷启动可发现性三路径"),
          "SIMILAR", "冷启动 残引号+模式 判同")
    check(titles_similar('离线引擎↔第一方 CLI 自动对账"模式', "离线与CLI自动对账模式"),
          "SIMILAR", "离线↔CLI对账 变体判同(3-gram)")
    check(not titles_similar("操作注意点应路由归位", "冷启动可发现性三路径"),
          "SIMILAR", "异卡判异(不误报)")

    # source_covering_cards: group sources subset of a reflect card's sources -> covered.
    fake_srcs = {"冷启动可发现性三路径": {"log-2026-09-27-06", "log-2026-09-27-07", "log-2026-09-28-01"}}
    g_cover = {"title": '冷启动可发现性三路径"模式', "sources": ["log-2026-09-27-06"]}
    g_new = {"title": "操作注意点应路由归位", "sources": ["log-2026-09-28-01"]}
    check(source_covering_cards(g_cover, fake_srcs) == ["冷启动可发现性三路径"], "COVER", "来源子集→命中已结晶卡")
    check(source_covering_cards(g_new, fake_srcs) == ["冷启动可发现性三路径"]
          and not titles_similar(g_new["title"], "冷启动可发现性三路径"),
          "COVER", "同来源但异标题→仅 maybe(不确信重复)")

    # Bug 4 regression: one source log backs two cards; the title-similar card must be chosen
    # as the confident duplicate, not whichever the dict order happens to return first.
    fake_srcs2 = {
        "冷启动可发现性三路径": {"log-2026-09-27-06", "log-2026-09-27-07", "log-2026-09-28-01"},
        "离线与CLI自动对账模式": {"log-2026-09-27-07", "log-2026-09-28-02"},
    }
    g_shared = {"title": '离线引擎↔第一方 CLI 自动对账"模式', "sources": ["log-2026-09-27-07"]}
    covers = source_covering_cards(g_shared, fake_srcs2)
    similar = [c for c in covers if titles_similar(g_shared["title"], c)]
    check(len(covers) == 2 and similar == ["离线与CLI自动对账模式"], "COVER",
          "共享来源多卡覆盖→选标题相似者为 crystallized_as(修 Bug4)")

    # Nested split: scan_logs must read 04-Logs/logs/ and skip corrections/.
    import tempfile, shutil
    tmp = tempfile.mkdtemp(prefix="reflect_selftest.")
    try:
        ld = os.path.join(tmp, "04-Logs", "logs")
        cd = os.path.join(tmp, "04-Logs", "corrections")
        os.makedirs(ld); os.makedirs(cd)
        with open(os.path.join(ld, "log-2026-09-27-06.md"), "w", encoding="utf-8") as f:
            f.write('---\nid: "log-2026-09-27-06"\n---\n\n## 候选结晶\n\n- 待结晶：甲 → procedure 卡。\n')
        with open(os.path.join(cd, "corr-2026-00001.md"), "w", encoding="utf-8") as f:
            f.write('---\nid: "corr-2026-00001"\n---\n\n## 候选结晶\n\n- 不应被扫到。\n')
        scanned = scan_logs(tmp)
        sids = {lg["log_id"] for lg in scanned}
        check(sids == {"log-2026-09-27-06"}, "NESTED",
              "scan_logs 读 logs/ 且跳过 corrections/ 实得%s" % sorted(sids))
        check(scanned and scanned[0]["path"] == "04-Logs/logs/log-2026-09-27-06.md",
              "NESTED", "path 含 logs/ 子目录")

        # G1-3 recursion fixtures: incubated cards (02-Cards/incubator/) must be
        # collected by both dedupe inputs; flat cards must keep being collected.
        flat_dir = os.path.join(tmp, "02-Cards")
        inc_dir = os.path.join(flat_dir, "incubator")
        os.makedirs(inc_dir)
        with open(os.path.join(flat_dir, "平铺卡.md"), "w", encoding="utf-8") as f:
            f.write('---\nid: "c-2026-00001"\nprovenance: reflect\n---\n\n'
                    '# 平铺卡\n\n来源 log-2026-09-27-06。\n')
        with open(os.path.join(inc_dir, "孵化卡.md"), "w", encoding="utf-8") as f:
            f.write('---\nid: "c-2026-00002"\nprovenance: reflect\n---\n\n'
                    '# 孵化卡\n\n来源 log-2026-09-28-01。\n')
        titles, ids = existing_card_titles(tmp)
        check("孵化卡" in titles and "c-2026-00002" in ids, "INCUBATOR",
              "incubator/ 卡标题+id 被 existing_card_titles 收集")
        check("平铺卡" in titles and "c-2026-00001" in ids, "INCUBATOR",
              "平铺卡照常收集（不回归）")
        srcs = reflect_card_sources(tmp)
        check(srcs.get("孵化卡") == {"log-2026-09-28-01"}, "INCUBATOR",
              "incubator/ reflect 卡来源被 reflect_card_sources 收集")
        check(srcs.get("平铺卡") == {"log-2026-09-27-06"}, "INCUBATOR",
              "平铺 reflect 卡来源照常收集（不回归）")
    finally:
        shutil.rmtree(tmp, ignore_errors=True)

    print("\nself-test: %s" % ("PASS" if ok else "FAIL"))
    return 0 if ok else 1


# ---------------------------------------------------------------- main

def main():
    ap = argparse.ArgumentParser()
    ap.add_argument("--vault")
    ap.add_argument("--json", action="store_true")
    ap.add_argument("--apply", help="落盘选中的 L2 工作单编号，逗号分隔，如 1,3")
    ap.add_argument("--force", action="store_true", help="越过'疑似已结晶'守卫强制落盘")
    ap.add_argument("--domain", help="按编号指定 domain，形如 1:memory-governance,3:knowledge-engineering")
    ap.add_argument("--date", help="覆盖 created/last-touched 日期（YYYY-MM-DD）")
    ap.add_argument("--self-test", action="store_true")
    a = ap.parse_args()

    if a.self_test:
        return self_test()

    root = vault_root(a.vault)
    if not os.path.isdir(root):
        print("vault 根不存在: %s" % root)
        return 2

    groups, l3, done = rollup(root)
    today = a.date or datetime.date.today().isoformat()

    if a.json:
        print(json.dumps({
            "tool": "reflect", "version": VERSION, "vault": root,
            "l2_groups": groups, "l3_manual_route": l3, "done_skipped": done,
        }, ensure_ascii=False, indent=2))
        return 0

    if not a.apply:
        print_worklist(groups, l3, done, root)
        print("")
        print("落盘：python 90-Meta/scripts/reflect.py --apply 编号,编号 [--domain 编号:域,...] [--force]")
        return 0

    picks = [int(tok.strip()) for tok in a.apply.split(",") if tok.strip().isdigit()]
    domain_map = {}
    if a.domain:
        for tok in a.domain.split(","):
            if ":" in tok:
                k, v = tok.split(":", 1)
                if k.strip().isdigit() and 1 <= int(k.strip()) <= len(groups):
                    domain_map[groups[int(k.strip()) - 1]["slug"]] = v.strip()
    written, backfilled, skipped = apply_groups(root, groups, picks, domain_map, today, a.force)
    print("reflect.py --apply 落盘完成：")
    for cid, name in written:
        print("  + 02-Cards/%s.md  (%s)" % (name, cid))
    for src, cid in backfilled:
        print("  ~ %s reflect-to += %s" % (src, cid))
    for n, title, why in skipped:
        print("  - 跳过 [%d] %s（%s）" % (n, title, why))
    print("")
    print("后续手工步骤（reflect 无权，须 Agent/人补）：")
    print("  1. 精修卡内容（定义/存疑/应用/边），summary 压到 ≤50 token；")
    print("  2. 把新卡加进 03-Atlas/{domain}/SH-* 的 auto 区块（否则 ROOT 不可达→UNREACHABLE）；")
    print("  3. 跑 entry_doctor 闸门（0 ERROR）后提交。")
    return 0


if __name__ == "__main__":
    try:
        sys.exit(main())
    except Exception as e:
        print("reflect.py 自身失败: %r" % e)
        sys.exit(2)
````
