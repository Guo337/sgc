---
id: vol-read-orchestrator
type: script-volume
created: 2026-11-16
last-touched: 2026-11-20
source-of: 90-Meta/scripts/read_orchestrator.py
fence: "````"
eol: crlf
depends-on: [vol-entry-doctor, vol-ttl-locator]
sha256: 2aa1d89bafcf93f7e663ad54436f66a2692581f39fd8ab0638ba273ef6623a36
---
````python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
"""
read_orchestrator.py — TTL P1 minimum native read orchestration (first slice)

Implements exactly the P1 first slice authorized by 02-STAGE-GATE §P1 + STATUS §6:

    probe -> ROOT L0 -> routing -> sub-entry -> one-hop read (target + links/backlinks)
          -> aliases / property filtering -> budget trimming -> explainable hit path
          -> baseline measurement

Design stance (respect SPEC-READ; do NOT create a second read system):
- The orchestration core consumes MCP `vault_read`-shaped read records
  ({path, content, frontmatter, links, backlinks, unresolvedLinks}). The executable
  backend in THIS slice is OfflineReader (SPEC-READ access-channel #3: grep/Python
  offline), which populates that same shape from disk. Wiring a live in-script MCP/REST
  or CLI client is deliberately OUT of this slice (it would risk a second access path /
  an MCP whitelist change); live MCP fidelity is verified separately by cross-checking a
  real `vault_read` links/backlinks against the offline reader (see the P1 verification
  log). The channel probe -> route -> degrade behaviour of SPEC-READ is still modelled
  and reported honestly (see resolve_channel).
- ONE hop only. NO PPR / BM25 / RRF ranking: those belong to Retrieval Stage 1 (query.py)
  = implementation phase P4, explicitly out of P1 scope. The routing full-text fallback is
  a plain substring scan, NOT a ranking model.
- Read-only. Writes no files, no network, no third-party deps. Stdlib only.
- Reuses entry_doctor's frontmatter / wikilink parsers so offline link semantics never
  drift from the doctor gate (identical strip_code + WIKILINK_RE + MDLINK resolution).

v0.3.0 overview category filter (overview internalization, corr-2026-11-20-89):
type:overview nodes (human-only README maps) never enter candidates / assembly /
one-hop neighbors — schema overview.exclusion.assembly=never, enforced here at four
points (_build_records / collect_candidates / full-text fallback / one-hop neighbors),
not merely declared (E principle).

Usage:
  python read_orchestrator.py --probe "EFG三原理"
  python read_orchestrator.py --probe "PPR" --domain knowledge-engineering --json
  python read_orchestrator.py --probe "..." --channel offline    # force offline backend
  python read_orchestrator.py --self-test                        # in-memory fixtures, no disk
  python read_orchestrator.py --vault PATH --probe "..."

Exit codes: 0 ran fine (hit OR reported gap) / 1 --self-test failed / 2 tool self-failure
NOTE: comments/docstrings are English by project convention; user-facing text stays
Chinese (the vault is a Chinese library).
"""
import sys, os, re, json, time, argparse, posixpath

# Reuse the doctor's battle-tested parsers so link semantics stay identical (no drift).
# depends-on: vol-entry-doctor, vol-ttl-locator (extract both before running)
sys.path.insert(0, os.path.dirname(os.path.abspath(__file__)))
from entry_doctor import (parse_fm, strip_code, as_name, load_vault, HARD_SKIP,
                          WIKILINK_RE, MDLINK_ANY_RE, H1_RE)
from ttl_locator import VaultResolutionError, resolve_vault

try:
    sys.stdout.reconfigure(encoding="utf-8")
except Exception:
    pass

VERSION = "0.3.0"
SLICE = "P1-one-hop"

# Budget defaults mirror schema.yaml `budgets:` (authoritative); used when schema unreadable
# or when running self-test in memory.
DEFAULT_BUDGETS = {"root_full": 2000, "root_L0": 500, "atlas_entry": 1000,
                   "card_summary": 50, "assembly": {"L3": 2000, "L2": 4000, "L1": 1000}}

# overview（全景概览）：纯人读 README 地图，永不进检索装配（schema overview.exclusion.
# assembly=never，corr-2026-11-20-89）。类别过滤是执行者（E 原理：非仅声明）——四处过滤：
# _build_records / collect_candidates / 全文兜底 / 一跳邻居。硬编码 type 集合先例=doctor
# GOV_TYPES（治理族过滤缺口系 P2 附带发现，scope 外只登记不修）。
OVERVIEW_TYPES = {"overview"}

CJK_RE = re.compile(r"[\u3400-\u4dbf\u4e00-\u9fff\uf900-\ufaff]")
LATIN_WORD_RE = re.compile(r"[A-Za-z0-9_]+")
EDGE_WL_RE = re.compile(r"\[\[([^\]]+)\]\]")
EDGE_REL_RE = re.compile(r"rel:\s*([A-Za-z_]+)")
EDGE_W_RE = re.compile(r"weight:\s*([0-9.]+)")


def approx_tokens(text):
    """Deterministic token approximation for baseline budgeting: 1 token per CJK char plus
    1 token per latin/digit word. This is a proxy (a real tokenizer is future work); it is
    stable and documented so budget trimming is reproducible."""
    if not text:
        return 0
    return len(CJK_RE.findall(text)) + len(LATIN_WORD_RE.findall(text))


TRIM_MARK = " …[trimmed]"


def truncate_to_tokens(text, cap):
    """Trim text so approx_tokens(result) <= cap INCLUDING the trim marker (binary search on
    characters). The marker's own token cost is reserved up front so the cap never overflows."""
    if cap <= 0:
        return ""
    if approx_tokens(text) <= cap:
        return text
    body_cap = max(0, cap - approx_tokens(TRIM_MARK))
    lo, hi = 0, len(text)
    while lo < hi:
        mid = (lo + hi + 1) // 2
        if approx_tokens(text[:mid]) <= body_cap:
            lo = mid
        else:
            hi = mid - 1
    return text[:lo].rstrip() + TRIM_MARK


def load_budgets(root):
    """Read the `budgets:` block from schema.yaml (single source of truth). Falls back to
    DEFAULT_BUDGETS with a note when schema is unreadable — never hard-fails."""
    if not root:
        return dict(DEFAULT_BUDGETS, assembly=dict(DEFAULT_BUDGETS["assembly"])), "无 vault 根，用默认预算"
    p = os.path.join(root, "90-Meta", "schema.yaml")
    if not os.path.isfile(p):
        return dict(DEFAULT_BUDGETS, assembly=dict(DEFAULT_BUDGETS["assembly"])), "schema.yaml 不存在，用默认预算"
    try:
        with open(p, encoding="utf-8") as fh:
            txt = fh.read()
    except Exception as e:
        return dict(DEFAULT_BUDGETS, assembly=dict(DEFAULT_BUDGETS["assembly"])), \
            "schema.yaml 读取失败(%s)，用默认预算" % type(e).__name__
    out = dict(DEFAULT_BUDGETS)
    out["assembly"] = dict(DEFAULT_BUDGETS["assembly"])
    m = re.search(r"(?m)^budgets:\s*(?:#.*)?$", txt)
    if not m:
        return out, "schema.yaml 未找到 budgets: 块，用默认预算"
    for ln in txt[m.end():].splitlines():
        if ln.strip() == "":
            continue
        if not ln[0].isspace():
            break                                   # next top-level key ends the block
        s = ln.split("#", 1)[0].strip()             # budget values never contain '#'
        mm = re.match(r"^(root_full|root_L0|atlas_entry|card_summary):\s*(\d+)", s)
        if mm:
            out[mm.group(1)] = int(mm.group(2))
        elif s.startswith("assembly:"):
            for k, v in re.findall(r"(L[123])\s*:\s*(\d+)", s):
                out["assembly"][k] = int(v)
    return out, None


# ---------------------------------------------------------------- offline reader

class OfflineReader:
    """SPEC-READ access channel #3 (offline grep/Python). Produces vault_read-shaped records:
    {path, content, frontmatter, links, backlinks, unresolvedLinks}. This is the sanctioned
    offline fallback, NOT a second read system and NOT build_graph.py (no persistence, no
    PPR/graph algorithm — just ephemeral one-hop link resolution for retrieval)."""
    channel = "offline"

    def __init__(self, files, extra=None, root=None, targets=None):
        self.files = files                          # {relpath(posix): text}
        self.extra = set(extra or ())
        self.root = root
        self.name_index = {}                        # as_name(rel).lower() -> rel
        self.alias_index = {}                       # alias.lower() -> rel
        self.records = {}                           # rel -> MCP-shaped record
        self.path_index = {}                        # lower(relpath) -> relpath (any legal target file)
        self.target_names = {}                      # lower(name or name.md) -> relpath
        for rel in files:
            self.name_index.setdefault(as_name(rel).lower(), rel)
        for tgt in (targets or ()):
            self._index_target(tgt.replace("\\", "/"))
        for name in self.extra:                     # from_files(): only bare names are known
            self.target_names.setdefault(name.lower(), name)
        self._build_records()
        self._build_links()

    def _index_target(self, rel):
        """Register one link-resolvable file (notes AND non-note targets such as schema.yaml
        or Templates/T-log.md — Obsidian resolves against every file on disk, not just notes)."""
        base = rel.split("/")[-1]
        self.path_index[rel.lower()] = rel
        self.target_names.setdefault(base.lower(), rel)
        if base.endswith(".md"):
            self.target_names.setdefault(base[:-3].lower(), rel)

    @classmethod
    def from_disk(cls, root):
        files, extra = load_vault(root)
        targets = []
        for dp, dn, fn in os.walk(root):
            dn[:] = [d for d in dn if d not in HARD_SKIP]
            for f in fn:
                targets.append(os.path.relpath(os.path.join(dp, f), root).replace("\\", "/"))
        return cls(files, extra, root, targets)

    @classmethod
    def from_files(cls, files, extra=None):
        return cls(files, extra)

    def _build_records(self):
        for rel, txt in self.files.items():
            fm, body = parse_fm(txt)
            fm = fm if isinstance(fm, dict) else {}
            # overview 类别过滤（1/4）：纯人读地图不建记录——候选/装配/邻居/反链
            # 全部消费 records，源头排除即全链排除（链接解析仍走 name_index，
            # 指向 overview 的链接可解析、只是永不装配）。
            if fm.get("type") in OVERVIEW_TYPES:
                continue
            h1 = H1_RE.search(body)
            aliases = fm.get("aliases")
            aliases = [a for a in aliases if isinstance(a, str)] if isinstance(aliases, list) else []
            dom = fm.get("domain")
            if isinstance(dom, list):
                dom = [str(d) for d in dom]
            elif dom:
                dom = [str(dom)]
            else:
                dom = []
            summ = fm.get("summary")
            self.records[rel] = {
                "path": rel, "content": txt, "frontmatter": fm, "body": body,
                "h1": h1.group(1).strip() if h1 else "",
                "summary": summ if isinstance(summ, str) else "",
                "aliases": aliases, "domain": dom,
                "type": fm.get("type"), "project": fm.get("project"), "id": fm.get("id"),
                "links": [], "backlinks": [], "unresolvedLinks": [],
            }
            for a in aliases:
                self.alias_index.setdefault(a.lower(), rel)

    def _resolve(self, link_text, base=None):
        raw = (link_text or "").strip().lstrip("/")
        if not raw:
            return None
        if base:
            joined = posixpath.normpath(posixpath.join(posixpath.dirname(base), raw))
            if joined.lower() in self.path_index:
                return self.path_index[joined.lower()]
        low = raw.lower()
        for cand in (low, low + ".md", low[:-3] if low.endswith(".md") else None):
            if cand and cand in self.path_index:
                return self.path_index[cand]
        t = raw.split("/")[-1]
        if t.endswith(".md"):
            t = t[:-3]
        key = t.lower()
        if key in self.name_index:
            return self.name_index[key]
        if key in self.alias_index:
            return self.alias_index[key]
        return self.target_names.get(key)

    def _build_links(self):
        for rel, rec in self.records.items():
            clean = strip_code(rec["content"])
            tgts, unres = [], []
            for wl in WIKILINK_RE.findall(clean):
                r = self._resolve(wl)
                if r and r != rel:
                    if r not in tgts:
                        tgts.append(r)
                elif not r:
                    w = wl.strip()
                    if w and w not in unres:
                        unres.append(w)
            for m in MDLINK_ANY_RE.findall(clean):
                m = m.strip()
                if not m or m.startswith("#") or re.match(r"^[a-zA-Z][a-zA-Z0-9+.\-]*:", m):
                    continue                        # anchor / external URL, not a vault link
                r = self._resolve(m, base=rel)
                if r and r != rel:
                    if r not in tgts:
                        tgts.append(r)
                elif r is None:
                    w = m.split("#")[0].strip()
                    if w and w not in unres:
                        unres.append(w)
            rec["links"] = tgts
            rec["unresolvedLinks"] = unres
        bl = {rel: [] for rel in self.records}
        for rel, rec in self.records.items():
            for t in rec["links"]:
                if t in bl and rel not in bl[t]:
                    bl[t].append(rel)
        for rel in self.records:
            self.records[rel]["backlinks"] = bl[rel]

    def read(self, rel):
        """vault_read-shaped record (throws KeyError if absent, mirroring MCP 'file not found')."""
        return self.records[rel]

    def list(self):
        """vault_list-shaped: sorted known paths."""
        return sorted(self.files)

    def search(self, query, limit=10):
        """search_simple-shaped fallback: plain substring scan (NOT BM25/PPR/RRF — those are
        Retrieval Stage 1 / query.py / P4, out of P1 scope). Returns [{filename, score}]."""
        q = (query or "").strip().lower()
        if not q:
            return []
        hits = []
        for rel, rec in self.records.items():
            if rec["type"] in OVERVIEW_TYPES:
                continue   # overview 类别过滤（3/4）：全文兜底也不得命中纯人读地图
            c = rec["content"].lower().count(q)
            if c:
                score = c
                if q in as_name(rel).lower():
                    score += 5
                if q in (rec["summary"] or "").lower():
                    score += 3
                hits.append({"filename": rel, "score": score})
        hits.sort(key=lambda x: (-x["score"], x["filename"]))
        return hits[:limit]


# ---------------------------------------------------------------- channel probe

def resolve_channel(pref):
    """Model SPEC-READ's probe -> route -> degrade. In this slice no live in-script MCP/CLI
    client is wired (out of scope; would risk a second read path / whitelist change), so the
    executable backend is offline channel #3. We report this honestly rather than pretending."""
    preferred = "mcp"                                # SPEC-READ read main channel
    if pref == "offline":
        return {"preferred": preferred, "used": "offline", "fallback_triggered": False,
                "reason": "显式离线通道（SPEC-READ 访问通道#3）"}
    return {"preferred": preferred, "used": "offline", "fallback_triggered": True,
            "reason": ("P1 第一切片未在脚本内接线 live MCP/REST 或 CLI 客户端（避免第二套读取通道 / "
                       "MCP 白名单变更）；按 SPEC-READ 探活→路由→降级 退离线通道#3。真实 MCP 保真度由"
                       "外部交叉验证（vault_read links/backlinks 对账）证明。")}


# ---------------------------------------------------------------- routing

def extract_root_l0(reader):
    """ROOT L0 = H1 + blockquotes before the first '##' + the 分区一览 partition list
    (SPEC-READ entry rule 1: 'H1 + blockquote + 分区标题', budget ~root_L0)."""
    rel = reader.name_index.get("00-root")
    if not rel:
        return "", None
    body = reader.records[rel]["body"]
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
            grab = ("分区一览" in st)
            if grab:
                out.append(ln.rstrip())
            continue
        if grab and st.startswith("- "):
            out.append(ln.rstrip())
    return "\n".join(out), rel


def forward_edges(rec):
    """Parse frontmatter k/v/e edges -> [{link, rel, weight}] (one-hop forward, annotated)."""
    fm = rec["frontmatter"]
    edges = []
    for key in ("k", "v", "e"):
        val = fm.get(key)
        if not isinstance(val, list):
            continue
        for item in val:
            if not isinstance(item, str):
                continue
            m = EDGE_WL_RE.search(item)
            if not m:
                continue
            rm = EDGE_REL_RE.search(item)
            wm = EDGE_W_RE.search(item)
            edges.append({"link": m.group(1).strip(),
                          "rel": rm.group(1) if rm else key,
                          "weight": float(wm.group(1)) if wm else None})
    return edges


def collect_candidates(probe, reader):
    """Stage-0 routing signals (no MAP.yaml — use native aliases per SPEC-READ rule 5).
    Returns [(level, rel, level_name, matched_token)]; level 1 = strongest."""
    p = (probe or "").strip()
    pl = p.lower()
    cands = []
    if not pl:
        return cands
    for rel, rec in reader.records.items():
        if rec["type"] in OVERVIEW_TYPES:
            continue   # overview 类别过滤（2/4）：永不进候选（防御性双保险）
        aliases_l = [a.lower() for a in rec["aliases"]]
        name = as_name(rel).lower()
        h1 = (rec["h1"] or "").lower()
        summ = (rec["summary"] or "").lower()
        if pl in aliases_l:
            cands.append((1, rel, "alias-exact", p))
        elif pl == name or pl == h1:
            cands.append((2, rel, "title-exact", p))
        elif pl in name or pl in h1:
            cands.append((3, rel, "title-substring", p))
        elif pl in summ:
            cands.append((4, rel, "summary-substring", p))
    return cands


def apply_filters(cands, reader, filters):
    if not filters:
        return cands, None
    out = []
    for c in cands:
        rec = reader.records[c[1]]
        fm = rec["frontmatter"]
        ok = True
        if filters.get("domain"):
            doms = [str(d).lower() for d in (rec["domain"] or [])]
            if str(filters["domain"]).lower() not in doms:
                ok = False
        if ok and filters.get("project"):
            if str(fm.get("project", "")).lower() != str(filters["project"]).lower():
                ok = False
        if ok and filters.get("type"):
            if str(fm.get("type", "")).lower() != str(filters["type"]).lower():
                ok = False
        if ok:
            out.append(c)
    note = None
    if cands and not out:
        note = "过滤条件 %s 排除了全部 %d 个候选" % (filters, len(cands))
    return out, note


def find_atlas(reader, domains):
    """SPEC-READ rule 3: a domain's 03-Atlas/SH-{domain}.md is the sub-entry, priority over
    the ROOT same-name partition."""
    for d in domains or []:
        rel = reader.name_index.get(("sh-%s" % d).lower())
        if rel:
            return rel, d
    return None, None


def atlas_entry_text(reader, atlas_rel, target_rel):
    rec = reader.records[atlas_rel]
    parts = []
    if rec["summary"]:
        parts.append(rec["summary"])
    for ln in rec["body"].splitlines():
        if ln.strip().startswith(">"):
            parts.append(ln.strip().lstrip(">").strip())
            break
    tname = as_name(target_rel)
    for ln in rec["body"].splitlines():
        if ("[[%s]]" % tname) in ln:
            parts.append(ln.strip())
            break
    return "\n".join(p for p in parts if p)


def build_entry_path(reader, target_rel, level, atlas_rel, domain):
    rec = reader.records[target_rel]
    name = as_name(target_rel)
    typ = rec["type"]
    path = ["00-ROOT L0"]
    if level == "fulltext-fallback":
        path.append("兜底链[全文/子串搜索]（路由/别名/标题均未命中，永不空手）")
    elif name == "00-ROOT":
        path.append("路由表[全局性/跨域综述] → ROOT 分区")
    elif typ == "spec" or name.startswith("SPEC-"):
        path.append("路由表[改 schema/规格/治理] → [[%s]]" % name)
    elif atlas_rel:
        path.append("路由表[特定域概念/机制] → SH-%s" % (domain or ""))
    else:
        path.append("路由表[特定域概念/机制/实体] → [[%s]]" % name)
    if atlas_rel:
        path.append("%s（子入口优先）" % atlas_rel)
    path.append(target_rel)
    return path


# ---------------------------------------------------------------- budget assembly

def assemble(reader, l0_text, atlas_text, target_rel, neighbor_rels, budgets):
    """Assemble the retrieval context under schema budgets. Trim order per SPEC-READ:
    L1 first -> L2 by relevance -> L3 (routing/red-lines) NOT trimmed."""
    trimmed = []
    l0_tokens = approx_tokens(l0_text)
    l0_over = l0_tokens > budgets["root_L0"]         # L3/L0 measured, not trimmed (spec: L3 不裁)

    atlas_tokens = 0
    if atlas_text:
        atlas_tokens = approx_tokens(atlas_text)
        if atlas_tokens > budgets["atlas_entry"]:
            atlas_text = truncate_to_tokens(atlas_text, budgets["atlas_entry"])
            atlas_tokens = budgets["atlas_entry"]
            trimmed.append("atlas_entry 超 %d token，已裁" % budgets["atlas_entry"])

    trec = reader.records[target_rel]
    tsum = trec["summary"] or ""
    if approx_tokens(tsum) > budgets["card_summary"]:
        tsum = truncate_to_tokens(tsum, budgets["card_summary"])
        trimmed.append("target summary 超 %d token，已裁" % budgets["card_summary"])
    tbody = trec["body"]

    neigh = []
    for nr in neighbor_rels:
        nrec = reader.records.get(nr)
        if not nrec:
            continue
        ns = nrec["summary"] or ""
        if approx_tokens(ns) > budgets["card_summary"]:
            ns = truncate_to_tokens(ns, budgets["card_summary"])
        neigh.append([nr, ns])

    l2cap = budgets["assembly"]["L2"]
    used = atlas_tokens + approx_tokens(tsum) + sum(approx_tokens(n[1]) for n in neigh)
    while neigh and used > l2cap:                     # drop farthest neighbor summaries first
        nr, ns = neigh.pop()
        used -= approx_tokens(ns)
        trimmed.append("丢弃邻居 summary %s（L2 预算）" % as_name(nr))
    body_budget = max(0, l2cap - used)
    if approx_tokens(tbody) > body_budget:
        tbody = truncate_to_tokens(tbody, body_budget)
        trimmed.append("target 正文裁至 L2 预算 %d token" % l2cap)
    used += approx_tokens(tbody)

    return {
        "caps": {"root_L0": budgets["root_L0"], "atlas_entry": budgets["atlas_entry"],
                 "card_summary": budgets["card_summary"], "assembly_L2": l2cap},
        "tokens": {"L3_L0": l0_tokens, "L0_over_budget": l0_over, "atlas": atlas_tokens,
                   "target_summary": approx_tokens(tsum), "target_body": approx_tokens(tbody),
                   "neighbors": sum(approx_tokens(n[1]) for n in neigh),
                   "L2_total": used, "tokens_assembled": l0_tokens + used},
        "trimmed": trimmed,
        "text": {"l0": l0_text, "atlas": atlas_text, "target_summary": tsum,
                 "target_body": tbody, "neighbors": [{"path": p, "summary": s} for p, s in neigh]},
    }


# ---------------------------------------------------------------- orchestration

def orchestrate(probe, reader, filters=None, channel_pref="auto", budgets=None):
    """Run the P1 first-slice pipeline; return a structured, explainable result dict."""
    t0 = time.time()
    filters = filters or {}
    if budgets is None:
        budgets, _ = load_budgets(reader.root)
    ch = resolve_channel(channel_pref)
    files_read = 0

    l0_text, root_rel = extract_root_l0(reader)
    if root_rel:
        files_read += 1

    cands = collect_candidates(probe, reader)
    cands, filter_note = apply_filters(cands, reader, filters)

    level, target_rel, why = None, None, None
    if cands:
        cands.sort(key=lambda c: (c[0], c[1]))
        level, target_rel, level_name, tok = cands[0]
        rec = reader.records[target_rel]
        why = "路由级别=%s：probe '%s' 命中 %s（id=%s）的 %s" % (
            level_name, tok, as_name(target_rel), rec["id"], level_name)
        level = level_name
    else:
        hits = reader.search(probe)                    # SPEC-READ fallback chain: never empty
        files_read += 1
        if hits:
            target_rel = hits[0]["filename"]
            level = "fulltext-fallback"
            why = "路由/别名/标题均未命中，退全文子串兜底：'%s' 命中 %s（score=%d）" % (
                probe, as_name(target_rel), hits[0]["score"])
        else:
            return {
                "tool": "read_orchestrator", "version": VERSION, "slice": SLICE,
                "probe": probe, "filters": filters, "status": "gap", "channel": ch,
                "route": None, "one_hop": None,
                "explain": {"what": "无命中", "why":
                            "库中无此知识（路由/别名/标题/全文兜底均空）。按 SPEC-READ 兜底链如实报告，"
                            "并按 [[SPEC-WRITE#记入协议]] 记录该缺口（缺口感知=建卡信号）。"
                            + (("；" + filter_note) if filter_note else ""),
                            "via": " → ".join(["00-ROOT L0", "兜底链[全文/子串搜索]", "空手而归→报缺口"])},
                "budget": None,
                "baseline": {"hops": 0, "files_read": files_read, "tokens_assembled": 0,
                             "hit_count": 0, "latency_ms": int((time.time() - t0) * 1000),
                             "channel": ch["used"]},
            }

    trec = reader.read(target_rel)
    files_read += 1
    atlas_rel, domain = find_atlas(reader, trec["domain"])
    atlas_text = ""
    if atlas_rel:
        atlas_text = atlas_entry_text(reader, atlas_rel, target_rel)
        files_read += 1

    # one hop: forward (edges + body mentions) + reverse (backlinks)
    fwd, seen = [], {target_rel}

    def _is_overview(r):
        # overview 类别过滤（4/4）：一跳邻居永不装配纯人读地图（前向+反向同判）。
        # 记录缺失≠安全：_build_records 已不建 overview 记录，但链接解析仍走
        # name_index（磁盘文件），须回读原文件判 type，否则边指向的 overview 漏网。
        nrec = reader.records.get(r)
        if nrec is not None:
            return nrec["type"] in OVERVIEW_TYPES
        txt = reader.files.get(r)
        if txt is None:
            return False
        fm, _ = parse_fm(txt)
        return isinstance(fm, dict) and fm.get("type") in OVERVIEW_TYPES

    for e in forward_edges(trec):
        r = reader._resolve(e["link"])
        if r and r != target_rel and r not in seen and not _is_overview(r):
            seen.add(r)
            fwd.append({"path": r, "rel": e["rel"], "weight": e["weight"]})
    for r in trec["links"]:
        if r != target_rel and r not in seen and not _is_overview(r):
            seen.add(r)
            fwd.append({"path": r, "rel": "mentions", "weight": None})
    rev = [{"path": b, "rel": "backlink"} for b in trec["backlinks"]
           if b != target_rel and not _is_overview(b)]

    neighbor_rels = [n["path"] for n in fwd] + [n["path"] for n in rev]
    budget = assemble(reader, l0_text, atlas_text, target_rel, neighbor_rels, budgets)
    files_read += len(budget["text"]["neighbors"])

    entry_path = build_entry_path(reader, target_rel, level, atlas_rel, domain)
    result = {
        "tool": "read_orchestrator", "version": VERSION, "slice": SLICE,
        "probe": probe, "filters": filters, "status": "hit", "channel": ch,
        "route": {"target": target_rel, "id": trec["id"], "type": trec["type"],
                  "domain": trec["domain"], "level": level, "why": why,
                  "entry_path": entry_path, "filter_note": filter_note,
                  "candidates": len(cands)},
        "one_hop": {"forward": fwd, "reverse": rev},
        "explain": {
            "what": "命中卡片 %s（id=%s, type=%s, domain=%s）" % (
                as_name(target_rel), trec["id"], trec["type"], ",".join(trec["domain"])),
            "why": why + (("；" + filter_note) if filter_note else ""),
            "via": " → ".join(entry_path),
        },
        "budget": budget,
        "baseline": {"hops": 1, "files_read": files_read,
                     "tokens_assembled": budget["tokens"]["tokens_assembled"],
                     "hit_count": 1 + len(fwd) + len(rev),
                     "latency_ms": int((time.time() - t0) * 1000),
                     "channel": ch["used"]},
    }
    return result


# ---------------------------------------------------------------- output

def output_human(r):
    print("read_orchestrator v%s — P1 一跳读取编排（%s）" % (VERSION, r["slice"]))
    print("probe: %s   filters: %s" % (r["probe"], r["filters"] or "无"))
    ch = r["channel"]
    print("通道: preferred=%s used=%s fallback=%s" % (
        ch["preferred"], ch["used"], "是" if ch["fallback_triggered"] else "否"))
    if ch["fallback_triggered"]:
        print("  降级原因: %s" % ch["reason"])
    print("状态: %s" % ("HIT" if r["status"] == "hit" else "GAP（报缺口，永不空手）"))
    ex = r["explain"]
    print("\n[可解释命中路径]")
    print("  what（命中什么）: %s" % ex["what"])
    print("  why （为什么命中）: %s" % ex["why"])
    print("  via （从哪个入口/路径）: %s" % ex["via"])
    if r["status"] == "hit":
        oh = r["one_hop"]
        fwd = ", ".join("%s[%s]" % (as_name(n["path"]), n["rel"]) for n in oh["forward"]) or "无"
        rev = ", ".join(as_name(n["path"]) for n in oh["reverse"]) or "无"
        print("\n[一跳] 前向: %s" % fwd)
        print("       反向(入边/backlinks): %s" % rev)
        b = r["budget"]
        tk = b["tokens"]
        print("\n[预算] L3/L0=%d(cap %d, over=%s)  atlas=%d  L2=%d/%d  assembled=%d" % (
            tk["L3_L0"], b["caps"]["root_L0"], "是" if tk["L0_over_budget"] else "否",
            tk["atlas"], tk["L2_total"], b["caps"]["assembly_L2"], tk["tokens_assembled"]))
        print("       裁剪: %s" % ("；".join(b["trimmed"]) if b["trimmed"] else "无"))
    bl = r["baseline"]
    print("\n[baseline] hops=%d files_read=%d tokens_assembled=%d hit_count=%d latency=%dms channel=%s" % (
        bl["hops"], bl["files_read"], bl["tokens_assembled"], bl["hit_count"],
        bl["latency_ms"], bl["channel"]))


# ---------------------------------------------------------------- self-test

def self_test():
    """In-memory fixtures (no disk writes): verify routing / one-hop / budget / explainability /
    gap / channel-fallback / property-filter / baseline. Mirrors entry_doctor's self-test style."""
    def card(cid, name, domain, aliases, edges, summary, body=""):
        al = ("aliases: %s\n" % json.dumps(aliases, ensure_ascii=False)) if aliases else ""
        ed = ""
        if edges:
            ed = "e:\n" + "".join(
                '  - { target: "[[%s]]", rel: %s, weight: %s }\n' % (t, rl, w) for t, rl, w in edges)
        return ("---\nid: \"%s\"\nlayer: 2\ntype: concept\nproject: memory-lib\ndomain: [%s]\n"
                "maturity: seed\ncreated: 2026-09-27\nlast-touched: 2026-09-27\nsummary: %s\n%s%s"
                "---\n# %s\n%s\n" % (cid, domain, summary, al, ed, name, body))

    files = {
        "00-ROOT.md": ("---\nid: \"spec-root-000\"\nlayer: 3\ntype: spec\nspec-kind: read\n"
                       "project: memory-lib\ndomain: [test]\nmaturity: growing\ncreated: 2026-09-27\n"
                       "last-touched: 2026-09-27\nsummary: root\n---\n# ROOT\n> 唯一根入口\n\n"
                       "## 分区一览\n\n- 入口 / 路由表\n\n## 入口\n\n- [SPEC-READ](01-Specs/SPEC-READ.md)\n"),
        "03-Atlas/SH-test.md": ("---\nid: \"s-1\"\nlayer: 2\ntype: summary\nproject: memory-lib\n"
                                "domain: [test]\nmaturity: seed\ncreated: 2026-09-27\nlast-touched: 2026-09-27\n"
                                "summary: 测试域分入口\nedge-policy: exempt\n---\n# SH-test\n> 小型提示：测试域导航\n\n"
                                "<!-- atlas:auto-start -->\n- [[alpha]] — 第一张卡\n- [[beta]] — 第二张卡\n"
                                "<!-- atlas:auto-end -->\n"),
        "02-Cards/alpha.md": card("c-a", "alpha", "test", ["阿尔法", "AlphaCard"],
                                  [("beta", "co_occurs", 0.5)], "alpha 卡", "正文提到 [[beta]]。"),
        "02-Cards/beta.md": card("c-b", "beta", "other", ["贝塔"],
                                 [("alpha", "co_occurs", 0.5)], "beta 卡"),
        "02-Cards/gamma.md": card("c-g", "gamma", "test", [], [], "gamma 卡",
                                  "长内容用于触发预算裁剪。" * 800),
    }
    reader = OfflineReader.from_files(files)
    budgets = dict(DEFAULT_BUDGETS, assembly=dict(DEFAULT_BUDGETS["assembly"]))
    checks = []

    def ck(name, cond):
        checks.append((name, bool(cond)))

    r = orchestrate("阿尔法", reader, budgets=budgets)
    ck("ROUTE_ALIAS", r["status"] == "hit" and r["route"]["target"] == "02-Cards/alpha.md"
       and r["route"]["level"] == "alias-exact")
    ck("ONE_HOP_FORWARD", any(n["path"] == "02-Cards/beta.md" for n in r["one_hop"]["forward"]))
    rb = orchestrate("beta", reader, budgets=budgets)
    ck("ONE_HOP_REVERSE", any(n["path"] == "02-Cards/alpha.md" for n in rb["one_hop"]["reverse"]))
    ck("BUDGET_CAP", r["budget"]["tokens"]["L2_total"] <= r["budget"]["caps"]["assembly_L2"])
    ck("EXPLAIN", all(r["explain"][k] for k in ("what", "why", "via")))
    rg = orchestrate("根本不存在的查询zzz", reader, budgets=budgets)
    ck("GAP", rg["status"] == "gap" and "记入协议" in rg["explain"]["why"])
    rc = orchestrate("alpha", reader, channel_pref="mcp", budgets=budgets)
    ck("CHANNEL_FALLBACK", rc["channel"]["used"] == "offline"
       and rc["channel"]["fallback_triggered"] is True)
    rf = orchestrate("卡", reader, filters={"domain": "test"}, budgets=budgets)
    ck("PROPERTY_FILTER", rf["status"] == "hit" and rf["route"]["target"] == "02-Cards/alpha.md")
    rt = orchestrate("gamma", reader, budgets=budgets)
    ck("BUDGET_TRIM", len(rt["budget"]["trimmed"]) > 0
       and rt["budget"]["tokens"]["L2_total"] <= rt["budget"]["caps"]["assembly_L2"])
    ck("BASELINE", r["baseline"]["hops"] == 1 and r["baseline"]["files_read"] > 0
       and r["baseline"]["tokens_assembled"] > 0)

    # overview 类别过滤（v0.3.0，corr-2026-11-20-89）：纯人读 README 永不进
    # 候选/装配/邻居。正例=overview 记录不建、探针不命中；反例=知识卡指向
    # overview 的边/正文链接与 overview 反向链均被邻居过滤兜住。
    ov_txt = ("---\nid: \"ov-2026-readme\"\ntype: overview\ncreated: 2026-11-20\n"
              "last-touched: 2026-11-20\nsummary: 全景地图 alpha 阿尔法\n"
              "aliases: [地图卡]\n---\n# README\n项目地图：[[alpha]] 与 [[beta]]。\n")
    ov_files = dict(files)
    ov_files["03-Atlas/README.md"] = ov_txt
    # delta：知识卡带指向 overview 的 e 边 + 正文提及（overview 也回链 delta → 反向过滤）
    ov_files["02-Cards/delta.md"] = card(
        "c-d", "delta", "test", [], [("README", "co_occurs", 0.5)],
        "delta 卡", "正文提到 [[README]]。")
    oreader = OfflineReader.from_files(ov_files)
    ck("OV_NO_RECORD", "03-Atlas/README.md" not in oreader.records)
    ck("OV_NOT_CANDIDATE",
       not any(rel == "03-Atlas/README.md" for _, rel, _, _ in
               collect_candidates("阿尔法", oreader))
       and not any(rel == "03-Atlas/README.md" for _, rel, _, _ in
                   collect_candidates("地图卡", oreader)))
    ck("OV_NOT_FALLBACK",
       not any(h["filename"] == "03-Atlas/README.md" for h in oreader.search("全景地图")))
    rov = orchestrate("delta", oreader, budgets=budgets)
    ck("OV_NOT_NEIGHBOR",
       rov["status"] == "hit" and rov["route"]["target"] == "02-Cards/delta.md"
       and not any(n["path"] == "03-Atlas/README.md" for n in rov["one_hop"]["forward"])
       and not any(n["path"] == "03-Atlas/README.md" for n in rov["one_hop"]["reverse"])
       and not any(n["path"] == "03-Atlas/README.md" for n in rov["budget"]["text"]["neighbors"]))
    # 原有卡不受影响（过滤不误伤知识条目）
    ra2 = orchestrate("阿尔法", oreader, budgets=budgets)
    ck("OV_KNOWLEDGE_INTACT", ra2["status"] == "hit"
       and ra2["route"]["target"] == "02-Cards/alpha.md")

    print("read_orchestrator --self-test（内置 in-memory fixture，未写盘）")
    ok = True
    for name, hit in checks:
        ok = ok and hit
        print("  [%s] %-18s %s" % ("OK" if hit else "MISS", name,
                                    "通过" if hit else "未通过（逻辑或 fixture 有误）"))
    print("\nself-test: %s（%d/%d 通过）" % ("PASS" if ok else "FAIL",
                                            sum(1 for _, h in checks if h), len(checks)))
    return 0 if ok else 1


# ---------------------------------------------------------------- main

def vault_root(cli):
    here = os.path.abspath(os.path.join(os.path.dirname(__file__), "..", ".."))
    location = resolve_vault(cli, script_fallback=here)
    return location.root, location.source


def main():
    ap = argparse.ArgumentParser(description="TTL P1 minimum native read orchestration (first slice)")
    ap.add_argument("--probe", help="retrieval probe (query string)")
    ap.add_argument("--domain")
    ap.add_argument("--project")
    ap.add_argument("--type", dest="typ")
    ap.add_argument("--channel", choices=["auto", "mcp", "cli", "offline"], default="auto")
    ap.add_argument("--vault")
    ap.add_argument("--json", action="store_true")
    ap.add_argument("--self-test", action="store_true")
    a = ap.parse_args()

    if a.self_test:
        return self_test()

    if not a.probe:
        print("需要 --probe（或用 --self-test）")
        return 2

    try:
        root, root_src = vault_root(a.vault)
    except VaultResolutionError as exc:
        print("vault 根解析失败: %s" % exc)
        return 2
    if not os.path.isdir(root):
        print("vault 根不存在: %s（来源: %s；用 --vault 或 $TTL_VAULT 指定 vault 根）"
              % (root, root_src))
        return 2

    reader = OfflineReader.from_disk(root)
    budgets, budget_note = load_budgets(root)
    filters = {k: v for k, v in {"domain": a.domain, "project": a.project, "type": a.typ}.items() if v}
    r = orchestrate(a.probe, reader, filters=filters, channel_pref=a.channel, budgets=budgets)
    if budget_note:
        r["budget_note"] = budget_note

    if a.json:
        print(json.dumps(r, ensure_ascii=False, indent=2))
    else:
        output_human(r)
    return 0


if __name__ == "__main__":
    try:
        sys.exit(main())
    except Exception as e:
        print("read_orchestrator 自身失败: %r" % e)
        sys.exit(2)
````
