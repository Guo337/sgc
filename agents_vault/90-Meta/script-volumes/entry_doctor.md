---
id: vol-entry-doctor
type: script-volume
created: 2026-11-16
last-touched: 2026-11-20
source-of: 90-Meta/scripts/entry_doctor.py
fence: "````"
eol: crlf
depends-on: [vol-ttl-locator]
sha256: d1387d53d089ee0b350e13a65336ee536af3ac41ccd132a8360824b78736a5f6
---
````python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
"""
entry_doctor.py — TTL memory-vault minimal read-only validator v2 (Stage 1 / P0 merge gate)

Implements the SPEC-GOVERN §doctor gate. Read-only scan of the vault's .md files (skipping
Templates/ etc.), validating schema v0.7 structure and governance constraints. Stdlib only;
writes no files, no network, no third-party deps.

v2.7.0 structural hardening (independent three-channel evaluation): parser layer rebuilt
(CommonMark-subset fence line-scanner; strict flat frontmatter subset, fail-loud outside
it; raw-byte sha256 unified with vol_extract); G20 shrunk to the sole VERSION anchor
(RULES/SELFTEST removed — decorative counts with forgeable truth channels).

v2.9.0 hub abstraction axis (batch 2B, corr-2026-11-16-58): hub:true L2 cards get
HUB_GROUNDS (in-degree>=2 strong edges) + HUB_DECOMPRESSION (abstracts edges must
have matching implements inverse); SECONDARY_TYPE (dual-type transition, DP-4).
abstracts rel added to schema rel_enum (implements inverse, decompression path).

v2.8.0 governance family (Agent-Governance internalization batch 1, corr-2026-11-16-57):
third node category alongside knowledge/volume — constraint/reference/status nodes get
GOV_* alternative criteria (G principle, volume precedent); closed-world two-way
exemption; ID_NAMESPACE exempts gov-*/vol-* (independent prefixes, federation boundary);
CONTRACT_DRIFT reconciles governance.required too. GOV_NAME deferred (zero-rename ruling).

v2.13.0 overview category (overview internalization, corr-2026-11-20-89): fourth node
category alongside knowledge/volume/governance — type:overview human-only README map
(03-Atlas/README.md) gets OV_* alternative criteria (G principle, volume/governance
precedent); closed-world two-way exemption; UNREACHABLE explicitly exempts overview
(human-only, deliberately off ROOT routing); BROKEN_LINK stays ERROR (no OV_LINK —
basic link honesty is not exempted); ID_NAMESPACE exempts ov-* (federation boundary);
CONTRACT_DRIFT reconciles overview.required too.

v2.14.0 pitfall trigger contract (nav wave 2/3 knife F1, PLAN-nav-wave23-closure):
TRIGGER_FM (ERROR) — type:pitfall cards must carry trigger (enum always/on-domain/
on-file-pattern/on-project, schema L3.trigger) and a non-empty scope. Runtime twin
of the session_start generator's fail-loud guard (CORRECT-1): the generator refuses
at injection time, TRIGGER_FM keeps the structure honest at scan time; neither waits
for the other. scope stays a non-empty check only (first version, no pattern grammar).

v2.15.0 atlas projection reconciliation (nav wave 2/3 knife F2, PLAN-nav-wave23-closure):
ATLAS_GROUP (WARN) — SH auto-block groups are mechanically reconciled against
hub.abstracts edges: hub-group members == abstracts targets ∩ C-1 set (multi-membership
cards must be listed in every group); unclustered == C-1 minus hub-group union minus
hub:true cards themselves (2A exemption, corr-93); C-1 full-set reconciliation == block
list incl. group heads (corr-84 catch mechanized). WARN not ERROR: the SH is a derived
view — a mismatch means rerun the generator (atlas_fill), not knowledge corruption.
Applicability is recognized from structure, not a hardcoded domain name: SH-{domain}
filename + auto block whose leading comment declares the C-1 rule (domain 含 X 的
02-Cards 卡, or a C-1 marker with X falling back to the filename domain segment);
first version supports the C-1 form only. Same finalized shape as the atlas_fill
v0.1.0 generator output (wave A, corr-95).

Design stance (borrow community-research methods; do NOT introduce a second authority):
- Broken links default to an offline regex (consistency first; strip code spans to match
  Obsidian's "no links inside code" semantics, D21). With --use-cli and obsidian on PATH,
  auto-reconcile `obsidian unresolved` against the offline broken set, classifying into
  agree / offline-false-positive / CLI-missed / non-wikilink target (dir / markdown links);
  genuine divergences escalate to WARN (decision B: do not rebuild the link engine).
- Every issue carries rule + fix (repair direction), meeting the P0 acceptance bar: the report
  must state "file + rule + fix direction".
- --self-test uses built-in in-memory fixtures (deliberately broken) to prove doctor catches
  them; it never writes to disk and never touches the real vault.

Usage:
  python entry_doctor.py                 # human-readable report
  python entry_doctor.py --json          # machine-readable (stable schema)
  python entry_doctor.py --vault PATH    # explicit vault root
  python entry_doctor.py --use-cli       # add obsidian unresolved auto-reconcile (degrades if App down)
  python entry_doctor.py --self-test     # run built-in destructive fixtures, verify each rule fires

Exit codes: 0 no ERROR / 1 has ERROR / 2 tool self-failure (--self-test failure also returns 1)
NOTE: comments/docstrings are English by project convention; user-facing report text stays
Chinese (the vault is a Chinese library).
"""
import sys, os, re, json, argparse, subprocess, hashlib
from datetime import date
from ttl_locator import VaultResolutionError, resolve_vault   # depends-on: vol-ttl-locator

try:
    sys.stdout.reconfigure(encoding="utf-8")
except Exception:
    pass

VERSION = "2.15.0"
STALE_DAYS = 180
SUMMARY_SOFT_CHARS = 120          # rough character proxy for the "summary <= 50 tokens" rule

# script-volume (脚本卷): heterogeneous node under the unified file interface (Unix
# "everything is a file" isomorph). NOT a knowledge entry: exempt from knowledge
# required fields / layer / type-layer checks; validated by VOL_* alternative criteria
# instead (G principle: exemption requires computable replacement criteria).
# Authority: schema.yaml script_volume section (corr-2026-11-16-48).
VOL_REQUIRED = ["id", "type", "created", "last-touched", "source-of", "fence", "eol", "sha256"]
VOL_ID_RE = re.compile(r"^vol-[a-z][a-z0-9-]*$")   # vol-{program}, collision-qualified vol-{dir}-{program} also legal
VOL_BLOCK_RE = re.compile(r"(?m)^(`{4,}|~{4,})([A-Za-z0-9_-]*)[^\r\n]*\r?\n(.*?)^\1[ \t]*\r?$", re.S)
# G20 DOC_IMPL_DRIFT: links carrying intent — [[volume#ANCHOR|alias]].
# The link itself declares "current-state reference" (plain text = historical, exempt).
# Anchor: VERSION (truth=code VERSION const, ERROR). v2.7.0 TCB shrink: RULES/SELFTEST
# anchors removed (decorative counts with forgeable truth channels; independent eval
# A+B converged). Anchor match is case-insensitive; alias optional (no-alias → WARN).
ANCHOR_LINK_RE = re.compile(r"\[\[([^\]\|#]+)#([A-Za-z_]+)(?:\|([^\]]*))?\]\]")
VOL_VERSION_RE = re.compile(r"(?m)^VERSION\s*=\s*[\"']([^\"']+)[\"']")
# version claim: full semver-ish token incl. prerelease/build suffix (B8: -beta is
# a different release, not the same version); multiple matches → ambiguous (S7).
CLAIMED_VER_RE = re.compile(r"v?(\d+(?:\.\d+)+(?:[-+][0-9A-Za-z.-]+)?)")

REQUIRED_COMMON = ["id", "layer", "type", "project", "domain",
                   "maturity", "created", "last-touched", "summary"]
TYPE_BY_LAYER = {
    1: {"log", "report", "correction"},
    2: {"concept", "claim", "procedure", "observation", "entity", "summary"},
    3: {"spec", "pitfall", "feedback"},
}
SPEC_KINDS = {"read", "write", "govern"}
# pitfall trigger legal enum (v2.14.0 TRIGGER_FM; truth source = schema.yaml
# L3.trigger). Same set as session_start.TRIGGER_ENUM (runtime twin, CORRECT-1).
TRIGGER_ENUM = {"always", "on-domain", "on-file-pattern", "on-project"}
# governance family (治理族): binding constraints / non-binding references / mutable
# status singleton, internalized from Agent-Governance/ (corr-2026-11-16-57). NOT
# knowledge entries: exempt from knowledge required fields / layer / type-layer checks;
# validated by GOV_* alternative criteria (G principle, volume precedent).
# Authority: schema.yaml governance section.
GOV_TYPES = {"constraint", "reference", "status"}
GOV_REQUIRED = ["id", "type", "created", "last-touched"]
GOV_ID_RE = re.compile(r"^gov-\d{4}-[a-z][a-z0-9-]*$")   # gov-{yyyy}-{slug}; singleton exception gov-status
# Knowledge-card fields a governance node must NOT carry (closed-world whitelist,
# volume precedent). aliases IS legal here (redirect/name-resolution layer, D0-1).
GOV_FORBIDDEN_FIELDS = ("layer", "k", "v", "e", "edge-policy", "project",
                        "domain", "maturity", "summary", "spec-kind",
                        "trigger", "session-task", "touched", "target", "hub")
# overview (全景概览): human-only project map (README), internalized as the fourth
# node category (corr-2026-11-20-89). NOT a knowledge entry: exempt from knowledge
# required fields / layer / type-layer checks; validated by OV_* alternative criteria
# (G principle, volume/governance precedent). Never assembled by the orchestrator.
# Authority: schema.yaml overview section.
OV_TYPES = {"overview"}
OV_REQUIRED = ["id", "type", "created", "last-touched", "summary"]
OV_ID_RE = re.compile(r"^ov-\d{4}-[a-z0-9-]+$")   # ov-{yyyy}-{slug}（如 ov-2026-readme）
# Knowledge-card fields an overview node must NOT carry (closed-world whitelist).
# summary IS required here (map self-description, schema overview.required);
# aliases IS legal (name-resolution layer, governance precedent).
OV_FORBIDDEN_FIELDS = ("layer", "k", "v", "e", "edge-policy", "project",
                       "domain", "maturity", "hub", "spec-kind",
                       "trigger", "session-task", "touched", "target")
# hub abstraction axis (v2.9.0, DP-5 方案A): strong semantic rels counted for
# in-degree; derives_from is "source" not "instance implementation", excluded.
HUB_STRONG_RELS = {"implements", "grounds", "specializes"}
# secondary-type legal enum (knowledge types only; governance types excluded —
# dual-type transition is for L2 knowledge cards, DP-4 再结晶-分化).
SECONDARY_TYPES = {"concept", "claim", "procedure", "observation", "entity",
                   "summary", "pitfall", "spec", "log", "report", "correction", "feedback"}
# v2.7.0 CONTRACT reconciliation (RC3): schema.yaml is the declared authority for
# these tables; the constants above are the implementation. G20 CONTRACT compares
# them at scan time — drift is ERROR, not silent second-authority accumulation.
SCHEMA_REL = "90-Meta/schema.yaml"
LOG_SECTIONS = ["## 做了什么", "## 结论与产出", "## 存疑与风险", "## 候选结晶"]
CORR_SECTIONS = ["## 改了什么", "## 为什么改", "## 依据"]
SKIP_DIRS = {"Templates", ".obsidian", ".git", "community-research",
             "node_modules", "__pycache__", ".trash", "inbox"}
# Hard skip: neither validated nor treated as a link target (VCS / config / third-party snapshots)
# inbox = NON-AUTHORITY upstream-submission workspace (sub-vault cards in transit,
# not formal vault content); excluded from validation and link-target set.
HARD_SKIP = {".obsidian", ".git", "community-research",
             "node_modules", "__pycache__", ".trash", "inbox"}
# Skip validation only, but still a legal link target (e.g. T-log/T-report under Templates)
VALIDATE_SKIP = SKIP_DIRS

WIKILINK_RE = re.compile(r"\[\[([^\]\|#]+)(?:#[^\]\|]*)?(?:\|[^\]]*)?\]\]")
MDLINK_RE = re.compile(r"\]\(([^)\s]+\.md)\)")
MDLINK_ANY_RE = re.compile(r"\]\(([^)\s]+)\)")
INLINE_CODE_RE = re.compile(r"`[^`\n]*`")
H1_RE = re.compile(r"(?m)^#\s+(.+?)\s*$")
AUTO_BLOCK_RE = re.compile(r"<!--\s*atlas:auto-start\s*-->(.*?)<!--\s*atlas:auto-end\s*-->", re.S)
# ATLAS_GROUP (v2.15.0, knife F2): SH domain-entry recognition + block line shapes.
# The C-1 domain comes from the block's leading rule comment (authority text, same
# source atlas_fill preserves); the SH-{domain} filename segment is the fallback.
# Nothing here hardcodes knowledge-engineering.
SH_NAME_RE = re.compile(r"^SH-(.+)$")
C1_COMMENT_RE = re.compile(r"domain\s*含\s*([A-Za-z0-9][A-Za-z0-9-]*)\s*的\s*02-Cards")
AG_LIST_LINE_RE = re.compile(r"^\s*-\s+\[\[([^\]\|#]+)")
AG_HUB_HEAD_RE = re.compile(r"^-\s+\[\[([^\]\|#]+)[^\]]*\]\]（枢纽）\s*$")
AG_UNC_HEAD_RE = re.compile(r"^-\s+未聚簇")

# CommonMark-subset fence scanner (v2.7.0, replaces the unanchored FENCE_RE regex).
# A regex cannot express fence nesting (context-free), so approximating it with
# (`{3,}).*?\1 both swallowed live links behind phantom inline "fences" (S2) and
# misjudged unclosed-at-EOF blocks (FP3). Line-scan rules:
#   - opener: <=3 leading spaces, 3+ backticks or tildes; backtick info string
#     must not contain a backtick (so ```a` is NOT a fence);
#   - closer: same char, length >= opener, nothing but spaces/tabs after it;
#   - unclosed fence extends to EOF (CommonMark), so its content stays code.
_FENCE_OPEN_RE = re.compile(r"^ {0,3}(`{3,}|~{3,})(.*)$")
_FENCE_CLOSE_RE = re.compile(r"^ {0,3}(`{3,}|~{3,})[ \t]*$")


def strip_fences(text):
    """Replace fenced code blocks with spaces, line-scanned per CommonMark subset."""
    out = []
    fence_ch, fence_len = None, 0
    for ln in text.splitlines():
        if fence_ch is None:
            m = _FENCE_OPEN_RE.match(ln)
            if m:
                marker, info = m.group(1), m.group(2)
                if marker[0] == "`" and "`" in info:
                    out.append(ln)          # info string has backtick: not a fence
                    continue
                fence_ch, fence_len = marker[0], len(marker)
                out.append("")
            else:
                out.append(ln)
        else:
            m = _FENCE_CLOSE_RE.match(ln)
            if m and m.group(1)[0] == fence_ch and len(m.group(1)) >= fence_len:
                fence_ch, fence_len = None, 0
            out.append("")
    return "\n".join(out)


def strip_comment(v):
    """Strip a trailing ' #...' comment from a value (# counts only outside quotes and after whitespace)."""
    out, inq, qch = [], False, ""
    for i, c in enumerate(v):
        if inq:
            out.append(c)
            if c == qch:
                inq = False
        elif c in "\"'":
            inq, qch = True, c
            out.append(c)
        elif c == "#" and (i == 0 or v[i - 1] in " \t"):
            break
        else:
            out.append(c)
    return "".join(out).strip()


def _scalar_unparseable(val):
    """True when a quoted scalar is unterminated or has trailing content after its closing
    quote (e.g. `summary: "a"尾巴`). Real YAML rejects both; Obsidian then drops the whole
    frontmatter, so its links/edges vanish from the index — offline must judge the same way."""
    if not val or val[0] not in "\"'":
        return False
    q, escaped = val[0], False
    for i in range(1, len(val)):
        ch = val[i]
        if escaped:
            escaped = False
        elif ch == "\\" and q == '"':
            escaped = True
        elif ch == q:
            return val[i + 1:].strip() != ""
    return True


def parse_fm(text):
    """Strict flat-subset frontmatter parse (v2.7.0): return (fm_dict, body);
    (None, text) when unparseable OR outside the declared subset.

    Subset (schema.yaml frontmatter_subset): flat `key: value` scalars, inline
    [a, b] lists, block lists of scalars or one-line flow maps ({...} kept as
    raw strings, as edges are consumed as text). Outside the subset — nested
    mappings, block scalars (> |), anchors/aliases, tabs, duplicate keys —
    real YAML/Obsidian either rejects or reinterprets the block, so doctor
    judges the whole frontmatter unparseable (FM_MISSING), same as Obsidian
    dropping it. Fail-loud beats silent approximation (S4/FP4)."""
    if not text.startswith("---"):
        return None, text
    m = re.match(r"^---\s*\n(.*?)\n---\s*\n?", text, re.S)
    if not m:
        return None, text
    block, body = m.group(1), text[m.end():]
    fm, lines, i = {}, block.splitlines(), 0
    while i < len(lines):
        ln = lines[i]
        if not ln.strip() or ln.lstrip().startswith("#"):
            i += 1
            continue
        if "\t" in ln:
            return None, text                 # YAML forbids tabs in indentation
        mm = re.match(r"^([A-Za-z0-9_\-]+):\s*(.*)$", ln)
        if not mm:
            if re.match(r"^\s+\S", ln):
                return None, text             # indented non-list line: nested map / continuation
            i += 1
            continue
        key, val = mm.group(1), mm.group(2)
        if key in fm:
            return None, text                 # duplicate key: real YAML rejects (S4)
        if val.strip() == "":                 # block list or empty value
            items, j = [], i + 1
            while j < len(lines) and re.match(r"^\s+-\s+", lines[j]):
                items.append(re.sub(r"^\s+-\s+", "", lines[j]).strip())
                j += 1
            fm[key] = items
            i = j if items else i + 1
            continue
        val = strip_comment(val)
        if val in (">", "|", ">-", "|-", ">+", "|+"):
            return None, text                 # block scalar: outside flat subset
        if val[:1] in ("&", "*"):
            return None, text                 # anchor / alias: outside flat subset
        if _scalar_unparseable(val):
            return None, text                 # 与 Obsidian 同判：整块 frontmatter 作废
        if val.startswith("[") and val.endswith("]"):
            inner = val[1:-1].strip()
            fm[key] = [x.strip().strip("\"'") for x in inner.split(",")] if inner else []
        else:
            sv = val.strip()
            if sv in ("true", "false"):
                # YAML boolean scalar (flat subset): unquoted true/false are
                # booleans, not strings. Quoted "false" stays a string (truthy
                # in YAML too). Fixes hub:false misjudged as hub (corr-2026-11-18-65).
                fm[key] = (sv == "true")
            else:
                fm[key] = sv.strip("\"'")
        i += 1
    return fm, body


def nonempty_edge(fm):
    for k in ("k", "v", "e"):
        v = fm.get(k)
        if isinstance(v, list) and v:
            return True
        if isinstance(v, str) and v.strip() and v.strip() != "[]":
            return True
    return False


def strip_code(text):
    """Strip fenced code blocks and inline code spans (match Obsidian: [[...]] inside code is not a link)."""
    text = strip_fences(text)
    text = INLINE_CODE_RE.sub(" ", text)
    return text


def _empty_required(v):
    """Required-field emptiness (S3): whitespace-only strings, None, the literal
    'null'/'~' (YAML null), and empty lists all count as missing for a required
    string field. Real YAML treats them as absent/empty; doctor must judge the same."""
    if v is None:
        return True
    if isinstance(v, list):
        return not v
    s = str(v).strip()
    return s == "" or s in ("null", "~")


def as_name(relpath):
    b = os.path.basename(relpath.replace("\\", "/"))
    return b[:-3] if b.endswith(".md") else b


def parse_date(v):
    if isinstance(v, str):
        try:
            return date.fromisoformat(v.strip()[:10])
        except Exception:
            return None
    return None


# ---------------------------------------------------------------- loading

def load_vault(root):
    """Walk the vault on disk: return (files, extra_names).
    files = {relpath(posix): text} to validate (VALIDATE_SKIP excluded);
    extra_names = names not validated but valid as link targets (e.g. Templates/T-log),
    to avoid false broken-link errors."""
    files, extra = {}, set()
    for dp, dn, fn in os.walk(root):
        dn[:] = [d for d in dn if d not in HARD_SKIP]
        rel_parts = os.path.relpath(dp, root).replace("\\", "/").split("/")
        skip_validate = any(p in VALIDATE_SKIP for p in rel_parts)
        for f in fn:
            if not f.endswith(".md"):
                continue
            full = os.path.join(dp, f)
            rel = os.path.relpath(full, root).replace("\\", "/")
            if skip_validate:
                extra.add(f)
                extra.add(f[:-3])
            else:
                # newline="" preserves the file's actual line endings: the sha256
                # integrity anchor hashes RAW bytes (v2.7.0, unified with
                # vol_extract), so universal-newline translation must not erase
                # the CRLF/LF distinction before hashing.
                with open(full, encoding="utf-8", newline="") as fh:
                    files[rel] = fh.read()
    return files, extra


# ---------------------------------------------------------------- reachability

def compute_reachable(files, texts, start="00-ROOT"):
    """BFS from start (default 00-ROOT): follow wikilink / .md links / directory links
    (prefix expansion). v2.8.0: start parameter lets GOV_ENTRY reuse the same BFS
    from the governance nav anchor (05-Governance/00-INDEX) during batches 1-3."""
    known = {as_name(rel): rel for rel in files}
    adj = {}
    for rel, txt in texts.items():
        name = as_name(rel)
        t = strip_code(txt)
        targets, dirs = set(), []
        for wl in WIKILINK_RE.findall(t):
            targets.add(wl.strip().split("/")[-1])
        for m in MDLINK_ANY_RE.findall(t):
            m = m.strip()
            if m.endswith(".md"):
                targets.add(as_name(m))
            elif m.endswith("/"):
                dirs.append(m.lstrip("./"))
        adj[name] = (targets, dirs)

    reachable, q = set(), ([start] if start in known else [])
    if start in known:
        reachable.add(start)
    while q:
        cur = q.pop()
        tg, dirs = adj.get(cur, (set(), []))
        for n in tg:
            if n in known and n not in reachable:
                reachable.add(n)
                q.append(n)
        for d in dirs:
            for rel2 in files:
                r2 = rel2.replace("\\", "/")
                if r2.startswith(d) or ("/" + d) in r2:
                    n2 = as_name(rel2)
                    if n2 not in reachable:
                        reachable.add(n2)
                        q.append(n2)
    return reachable, known


# ---------------------------------------------------------------- vault model

class VaultModel:
    """Read-only structured model built once from the vault files (the 'AST' of TTL).
    Rules consume this model; they never re-parse or share mutable local state."""

    def __init__(self, files, extra_resolvable=None, vault_root=None):
        self.files = files
        self.vault_root = vault_root   # disk root, for out-of-vault md-link existence checks
        self.parsed = {}
        self.resolvable = set(extra_resolvable or ())
        self.ids = {}
        self.titles = {}
        self.aliases = {}
        self.volumes = set()   # rel paths of script-volume nodes (heterogeneous, non-knowledge)
        self.governance = set()  # rel paths of governance-family nodes (constraint/reference/status)
        self.overview = set()  # rel paths of overview nodes (human-only README map, v2.13.0)
        self.vol_by_name = {}  # volume note name -> rel (for #VERSION link resolution)
        self.vol_name_dups = {}  # note name -> [rels] when >1 volume shares a basename (S5)
        self.vol_versions = {} # rel -> VERSION constant read from the code block (truth source)
        self.vol_by_id = {}    # vol id -> rel (dependency-graph primary key, VOL_DEPS)
        root_fm, _ = parse_fm(files.get("00-ROOT.md", ""))
        id_ns = root_fm.get("id-namespace") if isinstance(root_fm, dict) else None
        self.id_ns = id_ns.strip() or None if isinstance(id_ns, str) else None
        self.graph_texts = {}   # link-graph view: volume code blocks stripped
        for rel in sorted(files):
            txt = files[rel]
            fm, body = parse_fm(txt)
            self.parsed[rel] = (fm, body, txt)   # raw text kept for VOL_FENCE/SHA256
            b = os.path.basename(rel)
            self.resolvable.add(b)
            self.resolvable.add(b[:-3])
            # path-form wikilinks ([[dir/note]]) resolve in Obsidian; keep the
            # vault-relative path (sans .md) as a resolvable key too (FP1).
            if rel.endswith(".md"):
                self.resolvable.add(rel[:-3])
            gtxt = txt
            if isinstance(fm, dict):
                if fm.get("type") in GOV_TYPES:
                    self.governance.add(rel)
                if fm.get("type") in OV_TYPES:
                    self.overview.add(rel)
                if fm.get("type") == "script-volume":
                    self.volumes.add(rel)
                    nm = as_name(rel)
                    self.vol_name_dups.setdefault(nm, []).append(rel)
                    # first-seen wins; a shadow volume can no longer silently
                    # overwrite the truth source — VOL_NAME_DUP reports the
                    # collision (S5).
                    if nm not in self.vol_by_name:
                        self.vol_by_name[nm] = rel
                    vid = fm.get("id")
                    if isinstance(vid, str) and vid and vid not in self.vol_by_id:
                        self.vol_by_id[vid] = rel
                    blocks = _vol_blocks(txt)
                    if len(blocks) == 1:
                        mv = VOL_VERSION_RE.search(blocks[0][2])
                        if mv:
                            self.vol_versions[rel] = mv.group(1)
                    # code block is opaque program text: exclude from link-graph
                    # adjacency so fixture/format-template wikilinks don't create
                    # phantom edges (reachability must come from real prose links).
                    gtxt = VOL_BLOCK_RE.sub(" ", txt)
                al = fm.get("aliases")
                if isinstance(al, list):
                    self.resolvable.update(x for x in al if x)
            self.graph_texts[rel] = gtxt
        # Stateful aggregates: first-seen ownership (ID_DUP) and duplicate detection
        # (DUP_TITLE/DUP_ALIAS report rels[1:]) depend on sorted iteration order.
        for rel in sorted(files):
            fm, body, txt = self.parsed[rel]
            if not isinstance(fm, dict):
                continue
            _id = fm.get("id")
            if isinstance(_id, str) and _id and _id not in self.ids:
                self.ids[_id] = rel
            try:
                layer = int(fm.get("layer"))
            except (TypeError, ValueError):
                layer = None
            h1 = H1_RE.search(body)
            if h1 and layer in (2, 3):
                self.titles.setdefault(h1.group(1).strip(), []).append(rel)
            if isinstance(fm.get("aliases"), list):
                for a in fm["aliases"]:
                    if a:
                        self.aliases.setdefault(a, []).append(rel)
        self.reachable, self.known = compute_reachable(files, self.graph_texts)
        # v2.8.0 GOV_ENTRY transitional anchor: BFS from the governance nav page
        # (05-Governance/00-INDEX). Batch 4 (00-ENTRY internalized) switches the
        # anchor to the entry chain and upgrades GOV_ENTRY to ERROR.
        gov_nav = next((rel for rel in files
                        if rel.startswith("05-Governance/") and as_name(rel) == "00-INDEX"), None)
        self.gov_reachable = (compute_reachable(files, self.graph_texts, start=as_name(gov_nav))[0]
                              if gov_nav else set())
        # v2.7.0 category partition (RC4): computed once, consumed by the engine's
        # applicability dispatch. knowledge = fm-bearing non-volume; volume = volumes.
        # v2.8.0: governance is a third category, mutually exclusive with knowledge
        # (same as volumes) — knowledge rules must not leak onto governance nodes.
        # v2.13.0: overview is a fourth category, mutually exclusive too.
        self.knowledge = {rel for rel in files
                          if rel not in self.volumes and rel not in self.governance
                          and rel not in self.overview
                          and self.parsed[rel][0] is not None}
        # v2.9.0 hub axis: structured e-edge adjacency (target name -> [(rel, source rel)])
        # and abstracts edges for inverse reconciliation. e items are flow-map strings
        # kept as text by the flat parser; extract target/rel with regex.
        self.in_edges = {}    # target name -> list of (rel, source_rel)
        self.abstracts = {}   # source rel -> list of target names (abstracts edges)
        for rel in sorted(files):
            fm = self.parsed[rel][0]
            if not isinstance(fm, dict):
                continue
            ev = fm.get("e")
            if not isinstance(ev, list):
                continue
            for item in ev:
                if not isinstance(item, str):
                    continue
                mt = re.search(r"target:\s*\"?\[\[([^\]\|#]+)", item)
                mr = re.search(r"rel:\s*([a-z_]+)", item)
                if mt and mr:
                    tname = mt.group(1).strip().split("/")[-1]
                    rname = mr.group(1).strip()
                    self.in_edges.setdefault(tname, []).append((rname, rel))
                    if rname == "abstracts":
                        self.abstracts.setdefault(rel, []).append(tname)
        self.superseded = set()
        for rel in files:
            fm = self.parsed[rel][0]
            if not isinstance(fm, dict):
                continue
            ev = fm.get("e")
            if isinstance(ev, list):
                for item in ev:
                    if isinstance(item, str) and "supersedes" in item:
                        m = re.search(r"\[\[([^\]]+)\]\]", item)
                        if m:
                            self.superseded.add(m.group(1).strip().split("/")[-1])


# ---------------------------------------------------------------- rules
# Each rule is a stateless unit: fn(model, report) reads the model and calls report.
# scope: file (per-file) | link (per-link) | graph (whole graph) | cross (cross-source).
# applies: which file categories the engine feeds to the rule (v2.7.0, RC4 —
# "parse, don't validate": applicability is declared once and enforced centrally,
# so a forgotten per-rule volume guard can no longer leak knowledge assumptions
# onto volumes, A's one-way-exemption finding). Categories:
#   knowledge = frontmatter-bearing non-volume non-governance non-overview notes;
#   volume = script-volume nodes; governance = constraint/reference/status nodes
#   (v2.8.0); overview = human-only README map nodes (v2.13.0);
#   all = every scanned file (incl. frontmatter-less ones, for FM_MISSING).

class Rule:
    __slots__ = ("id", "scope", "fn", "level", "since", "applies")

    def __init__(self, id, scope, fn, level="ERROR", since="", applies=("all",)):
        self.id, self.scope, self.fn = id, scope, fn
        self.level, self.since, self.applies = level, since, applies


def rule_fm_missing(model, report):
    for rel in sorted(model.files):
        if model.parsed[rel][0] is None:
            report(rel, "ERROR", "FM_MISSING", "frontmatter 缺失或不可解析",
                   "在文件顶部补 --- 包裹的 YAML frontmatter（见 schema common）")


def rule_field_missing(model, report):
    for rel in sorted(model.files):
        fm = model.parsed[rel][0]
        if fm is None or rel in model.volumes:
            continue
        for k in REQUIRED_COMMON:
            if k not in fm or _empty_required(fm[k]):
                report(rel, "ERROR", "FIELD_MISSING", "缺必填字段: %s" % k,
                       "按 schema.yaml common 补字段 %s" % k)


def rule_id_dup(model, report):
    seen = {}
    for rel in sorted(model.files):
        fm = model.parsed[rel][0]
        if fm is None:
            continue
        _id = fm.get("id")
        if isinstance(_id, str) and _id:
            if _id in seen:
                report(rel, "ERROR", "ID_DUP", "id 重复: %s (与 %s)" % (_id, seen[_id]),
                       "改 id 保证全局唯一（见 id_format）")
            else:
                seen[_id] = rel


def rule_id_namespace(model, report):
    if not model.id_ns:
        return
    for rel in sorted(model.files):
        fm = model.parsed[rel][0]
        if fm is None:
            continue
        _id = fm.get("id")
        if rel != "00-ROOT.md" and isinstance(_id, str) and _id:
            kind = _id.split("-", 1)[0]
            if kind in ("gov", "vol", "ov"):
                # governance family / script volumes / overview carry independent id
                # prefixes (not knowledge {kind}-{slug}); federation boundary: a
                # sub-vault doctor seeing them judges "not this vault's ontology",
                # not an id-namespace violation (corr-2026-11-16-57; ov-* added
                # corr-2026-11-20-89).
                continue
            local_prefix = "%s-%s-" % (kind, model.id_ns)
            if not _id.startswith(local_prefix) and fm.get("provenance") != "copied":
                report(rel, "ERROR", "ID_NAMESPACE",
                       "id 未用本库命名空间前缀: %s（应为 %s…）" % (_id, local_prefix),
                       "新建条目改用本库前缀 %s…；若是赋能复制的上游条目，保留原 id 并标 provenance: copied（见 schema id_namespace）" % local_prefix)
            elif not _id.startswith(local_prefix) and fm.get("provenance") == "copied":
                # S6: copied exemption needs computable corroboration — a source
                # pointer to the upstream entry. Without it the exemption is a
                # one-field bypass; WARN (not ERROR) so real federation copies
                # are not blocked, but the missing provenance is visible.
                src = fm.get("source")
                if not (isinstance(src, str) and src.strip()):
                    report(rel, "WARN", "ID_NAMESPACE",
                           "copied 豁免缺来源佐证: %s 无 source 指针" % _id,
                           "补 source: 上游条目指针（如 [[上游库/卡]] 或上游 id），证明 copied 身份")


def rule_layer_bad(model, report):
    for rel in sorted(model.files):
        fm = model.parsed[rel][0]
        if fm is None or rel in model.volumes:
            continue
        try:
            int(fm.get("layer"))
        except (TypeError, ValueError):
            report(rel, "ERROR", "LAYER_BAD", "layer 非法: %r" % fm.get("layer"),
                   "layer 取 1/2/3")


def rule_type_layer(model, report):
    for rel in sorted(model.files):
        fm = model.parsed[rel][0]
        if fm is None or rel in model.volumes:
            continue
        try:
            layer = int(fm.get("layer"))
        except (TypeError, ValueError):
            layer = None
        typ = fm.get("type")
        if layer in TYPE_BY_LAYER and isinstance(typ, str) and typ not in TYPE_BY_LAYER[layer]:
            report(rel, "ERROR", "TYPE_LAYER",
                   "type↔layer 非法: L%s 不允许 type=%s" % (layer, typ),
                   "调整 type 或 layer：L1→log/report/correction；L2→concept/claim/procedure/observation/entity/summary；L3→spec/pitfall/feedback")


def rule_spec_kind(model, report):
    for rel in sorted(model.files):
        fm = model.parsed[rel][0]
        if fm is None:
            continue
        if fm.get("type") == "spec" and fm.get("spec-kind") not in SPEC_KINDS:
            report(rel, "ERROR", "SPEC_KIND", "spec 缺合法 spec-kind: %r" % fm.get("spec-kind"),
                   "type:spec 须带 spec-kind ∈ [read, write, govern]")



def rule_trigger_fm(model, report):
    """TRIGGER_FM (v2.14.0, knife F1): type:pitfall cards must declare a legal
    trigger enum and a non-empty scope (schema L3.trigger/scope). Missing or
    illegal fields make the SessionStart injection and the on-domain self-check
    silently skip the pitfall — ERROR here keeps the contract executable (E).
    scope is only checked non-empty in this first version (no pattern grammar)."""
    for rel in sorted(model.files):
        fm = model.parsed[rel][0]
        if not isinstance(fm, dict) or fm.get("type") != "pitfall":
            continue
        trig = fm.get("trigger")
        if trig is None or _empty_required(trig):
            report(rel, "ERROR", "TRIGGER_FM", "pitfall 缺 trigger",
                   "补 trigger ∈ [always, on-domain, on-file-pattern, on-project]（schema L3.trigger）")
        elif str(trig).strip() not in TRIGGER_ENUM:
            report(rel, "ERROR", "TRIGGER_FM", "pitfall trigger 非法枚举: %r" % trig,
                   "trigger 取 always/on-domain/on-file-pattern/on-project（schema L3.trigger）")
        scope = fm.get("scope")
        if scope is None or _empty_required(scope):
            report(rel, "ERROR", "TRIGGER_FM", "pitfall 缺 scope（或为空）",
                   "补非空 scope（trigger 的匹配范围：域名/文件模式/项目名，schema L3.scope）")


def rule_summary_long(model, report):
    for rel in sorted(model.files):
        fm = model.parsed[rel][0]
        if fm is None or rel in model.volumes:
            continue
        s = fm.get("summary")
        if isinstance(s, str) and len(s) > SUMMARY_SOFT_CHARS:
            report(rel, "WARN", "SUMMARY_LONG", "summary 偏长(%d 字符)" % len(s),
                   "压缩 summary 到 ≤50 token（自包含即可）")


def rule_atlas_empty(model, report):
    for rel in sorted(model.files):
        fm, body, txt = model.parsed[rel]
        if fm is None:
            continue
        try:
            layer = int(fm.get("layer"))
        except (TypeError, ValueError):
            layer = None
        if layer == 2 and fm.get("edge-policy") == "exempt":
            mb = AUTO_BLOCK_RE.search(body)
            inner = mb.group(1) if mb else ""
            links = WIKILINK_RE.findall(strip_code(inner)) if mb else []
            if not mb:
                report(rel, "ERROR", "ATLAS_EMPTY",
                       "edge-policy:exempt 但缺 atlas:auto 区块",
                       "exempt 导航节点须含 <!-- atlas:auto-start/end --> 区块并列出域内卡")
            elif not links:
                report(rel, "ERROR", "ATLAS_EMPTY",
                       "edge-policy:exempt 但 auto 区块内无卡片链接",
                       "在 atlas:auto 区块内补 ≥1 个 [[卡片]] 导航链接")


def _ag_domain_has(fm, domain):
    """C-1 membership predicate — same semantics as atlas_fill.domain_has
    (inline [a, b] list or comma string; both come from the flat parser)."""
    dom = fm.get("domain")
    if isinstance(dom, list):
        return domain in [str(d).strip() for d in dom]
    if isinstance(dom, str):
        return domain in [d.strip() for d in dom.strip("[]").split(",")]
    return False


def rule_atlas_group(model, report):
    """ATLAS_GROUP (v2.15.0, knife F2): reconcile the SH auto-block projection
    against hub.abstracts edges — the standing mechanical guard for the atlas_fill
    generator's output shape (same corr-finalized shape, wave A). Three checks:
      (1) each hub group's listed members == that hub's abstracts targets ∩ C-1
          (multi-membership cards must appear in EVERY group);
      (2) unclustered == C-1 minus all hub-group members minus hub:true cards
          themselves (Owner ruling 2A, corr-2026-11-20-93);
      (3) C-1 full set == the block's whole list incl. group heads (corr-84
          under-listing catch, mechanized).
    WARN not ERROR (PLAN): the SH is a derived view; a mismatch means the
    generator must be rerun, not that knowledge is corrupted. Applicability is
    structural: SH-{domain} filename + auto block whose leading comment declares
    the C-1 rule (contains the 'domain 含 X 的 02-Cards' pattern or a C-1 marker);
    X comes from the comment, falling back to the filename domain segment when
    the comment declares C-1 but the pattern does not parse. First version
    supports the C-1 form only — SH blocks without a C-1 comment (SH-index,
    role files) are out of scope and never reconciled here."""
    fix = "跑 atlas_fill --vault <库根> 核对 diff 后 --apply（SH 是派生视图，失配=重跑生成器）"
    for rel in sorted(model.files):
        fm, body, txt = model.parsed[rel]
        if not isinstance(fm, dict):
            continue
        mn = SH_NAME_RE.match(as_name(rel))
        if not mn:
            continue
        mb = AUTO_BLOCK_RE.search(body)
        if not mb:
            continue   # missing block is ATLAS_EMPTY's report, not ours
        inner = mb.group(1)
        lines = [ln.rstrip("\r") for ln in inner.split("\n")]
        # leading comment = run of non-list lines before the first list line
        # (same extraction as atlas_fill.leading_comment_lines)
        first_list = next((i for i, ln in enumerate(lines)
                           if AG_LIST_LINE_RE.match(ln)), None)
        if first_list is None:
            continue   # no list lines: ATLAS_EMPTY owns that report
        comment = "\n".join(lines[:first_list])
        mc = C1_COMMENT_RE.search(comment)
        if not mc and "C-1" not in comment:
            continue   # no C-1 declaration in the leading comment: out of scope
        # X from the comment pattern; filename domain segment is the fallback
        # when C-1 is declared but the pattern does not parse (first version).
        domain = mc.group(1) if mc else mn.group(1)
        # C-1 set from the parsed model (recursive 02-Cards, incubator included)
        members = {}
        for crel in sorted(model.files):
            if not crel.startswith("02-Cards/"):
                continue
            cfm = model.parsed[crel][0]
            if isinstance(cfm, dict) and _ag_domain_has(cfm, domain):
                members[as_name(crel)] = crel
        # hub:true judgment `is True` mirrors atlas_fill exactly (boolean scalar,
        # v2.12.0 parser; quoted "true" strings are a frontmatter-form issue).
        hub_names = {n for n in members if model.parsed[members[n]][0].get("hub") is True}
        # parse block structure: hub groups / unclustered section
        groups = []            # [(hub_name, [member names])]
        unclustered = []
        listed_all = set()
        cur = None
        in_unc = False
        for ln in lines[first_list:]:
            mh = AG_HUB_HEAD_RE.match(ln)
            if mh:
                cur = (mh.group(1).strip(), [])
                groups.append(cur)
                listed_all.add(cur[0])
                in_unc = False
                continue
            if AG_UNC_HEAD_RE.match(ln):
                in_unc = True
                cur = None
                continue
            ml = AG_LIST_LINE_RE.match(ln)
            if ml:
                name = ml.group(1).strip()
                listed_all.add(name)
                if in_unc:
                    unclustered.append(name)
                elif cur is not None:
                    cur[1].append(name)
        # (1) per-hub group reconciliation
        grouped_union = set()
        for name, listed in groups:
            grouped_union.update(listed)
            hrel = members.get(name)
            if hrel is None:
                report(rel, "WARN", "ATLAS_GROUP",
                       "枢纽组头 [[%s]] 不在 C-1 收录集（domain 含 %s）" % (name, domain), fix)
                continue
            expected = []
            for tgt in model.abstracts.get(hrel, []):
                if tgt in members and tgt not in expected:
                    expected.append(tgt)
            for tgt in expected:
                if tgt not in listed:
                    report(rel, "WARN", "ATLAS_GROUP",
                           "枢纽 [[%s]] 组缺卡 [[%s]]（abstracts 边∩C-1 应列，多归属须每组都列）" % (name, tgt), fix)
            for tgt in listed:
                if tgt not in expected:
                    report(rel, "WARN", "ATLAS_GROUP",
                           "枢纽 [[%s]] 组多列 [[%s]]（无对应 abstracts 边或不在 C-1）" % (name, tgt), fix)
        for name in sorted(hub_names):
            if name not in {g[0] for g in groups}:
                report(rel, "WARN", "ATLAS_GROUP",
                       "hub:true 卡 [[%s]] 缺枢纽组头" % name, fix)
        # (2) unclustered reconciliation (2A: hub cards themselves exempt)
        unc_expected = {n for n in members if n not in grouped_union and n not in hub_names}
        for n in unclustered:
            if n in hub_names or n in grouped_union:
                report(rel, "WARN", "ATLAS_GROUP",
                       "未聚簇组混入枢纽卡 [[%s]]（hub 组头或组成员，2A 豁免/应挂组）" % n, fix)
            elif n not in unc_expected:
                report(rel, "WARN", "ATLAS_GROUP",
                       "未聚簇组多列 [[%s]]（不在 C-1 收录集）" % n, fix)
        for n in sorted(unc_expected - set(unclustered)):
            report(rel, "WARN", "ATLAS_GROUP",
                   "未聚簇组缺卡 [[%s]]（C-1 内无枢纽归属应列）" % n, fix)
        # (3) C-1 full-set reconciliation (corr-84 catch mechanized)
        for n in sorted(set(members) - listed_all):
            report(rel, "WARN", "ATLAS_GROUP",
                   "C-1 漏收 [[%s]]（domain 含 %s 但清单未列）" % (n, domain), fix)
        for n in sorted(listed_all - set(members)):
            report(rel, "WARN", "ATLAS_GROUP",
                   "C-1 多收 [[%s]]（清单已列但 domain 不含 %s 或卡不存在）" % (n, domain), fix)


def rule_edge_orphan(model, report):
    for rel in sorted(model.files):
        fm = model.parsed[rel][0]
        if fm is None:
            continue
        try:
            layer = int(fm.get("layer"))
        except (TypeError, ValueError):
            layer = None
        if layer == 2 and fm.get("edge-policy") != "exempt" and not nonempty_edge(fm):
            report(rel, "ERROR", "EDGE_ORPHAN",
                   "L2 卡无边(k/v/e 皆空)且未设 edge-policy:exempt → 孤岛",
                   "补至少一条 k/v/e 边；若确为 Atlas 导航节点则设 edge-policy:exempt")


def rule_log_section(model, report):
    for rel in sorted(model.files):
        fm, body, txt = model.parsed[rel]
        if fm is None:
            continue
        if fm.get("type") == "log":
            for sec in LOG_SECTIONS:
                if sec not in body:
                    report(rel, "ERROR", "LOG_SECTION", "日志缺固定段: %s" % sec,
                           "补齐日志四固定段：做了什么/结论与产出/存疑与风险/候选结晶")


def rule_corr_section(model, report):
    for rel in sorted(model.files):
        fm, body, txt = model.parsed[rel]
        if fm is None:
            continue
        if fm.get("type") == "correction":
            for sec in CORR_SECTIONS:
                if sec not in body:
                    report(rel, "ERROR", "CORR_SECTION", "更正说明缺固定段: %s" % sec,
                           "补齐更正说明三段：改了什么/为什么改（原来错在哪）/依据（模板 T-correction）")


def rule_corr_target(model, report):
    for rel in sorted(model.files):
        fm = model.parsed[rel][0]
        if fm is None:
            continue
        if fm.get("type") == "correction":
            tgt = fm.get("target")
            if not tgt or not str(tgt).strip():
                report(rel, "ERROR", "CORR_TARGET", "更正说明缺 target（被更正规格/卡）",
                       "frontmatter 补 target: \"[[SPEC-…]]\"（必填）")


def rule_spec_changelog(model, report):
    for rel in sorted(model.files):
        fm, body, txt = model.parsed[rel]
        if fm is None:
            continue
        if fm.get("type") == "spec" and re.search(r"^##\s+changelog\s*$", body, re.M | re.I):
            report(rel, "ERROR", "SPEC_CHANGELOG", "规格内嵌 changelog 实体段（应下沉）",
                   "修订说明下沉至 04-Logs/corrections/（type:correction），规格只留当前态 + 「修订说明（已下沉）」指针节")


def rule_broken_link(model, report):
    for rel in sorted(model.files):
        fm, body, txt = model.parsed[rel]
        if fm is None:
            continue
        if rel in model.volumes:
            # volume code block is opaque program text, not vault prose: its
            # wikilink-shaped strings (fixtures, format templates) are not links.
            txt = model.graph_texts[rel]
        stripped = strip_code(txt)
        for tgt in WIKILINK_RE.findall(stripped):
            t = tgt.strip()
            if t and t not in model.resolvable:
                report(rel, "ERROR", "BROKEN_LINK", "断链 [[%s]] 无法解析" % t,
                       "建目标笔记，或修正/移除该链接（离线判定；--use-cli 可自动对账）")
        # markdown .md links are live links in Obsidian too (S8): offline must
        # check their reachability, not only wikilinks. External URLs / non-md
        # targets stay out of scope (CLI reconcile classifies them). Links that
        # escape the vault root (../ beyond root, e.g. project-appendage Audit/
        # docs) are out of the vault link graph — existence-checked on disk
        # instead, since Obsidian cannot resolve them either way.
        for tgt in MDLINK_RE.findall(stripped):
            t = tgt.strip()
            if t.startswith(("http://", "https://", "file://", "#")):
                continue
            path_part = t.split("#")[0]
            norm = os.path.normpath(os.path.join(os.path.dirname(rel), path_part))
            if norm.startswith(".."):
                if model.vault_root:
                    full = os.path.join(model.vault_root, norm)
                    if not os.path.isfile(full):
                        report(rel, "ERROR", "BROKEN_LINK",
                               "断链 [md](%s) 库外目标不存在" % t,
                               "修正库外路径，或改为纯文本引用（库外文档不参与库内链接图）")
                continue
            name = as_name(path_part)
            if name and name not in model.resolvable:
                report(rel, "ERROR", "BROKEN_LINK", "断链 [md](%s) 无法解析" % t,
                       "建目标笔记，或修正/移除该链接（markdown 形态同样校验）")


def rule_unreachable(model, report):
    # overview 显式豁免（v2.13.0）：纯人读故意不挂 ROOT 路由，不豁免则恒报 WARN
    # 自绊 0E/0W/0I 验收（schema overview 节，corr-2026-11-20-89）。
    exempt_types = {"log", "report", "correction", "pitfall", "overview"}
    for rel in sorted(model.files):
        name = as_name(rel)
        fm = model.parsed[rel][0]
        typ = fm.get("type") if isinstance(fm, dict) else None
        if name == "00-ROOT" or typ in exempt_types:
            continue
        if name not in model.reachable:
            report(rel, "WARN", "UNREACHABLE", "从 00-ROOT 路由不可达",
                   "从 ROOT/对应 SH-Atlas/相关卡建立指向本文件的链接（pitfall/log/report/correction/overview 已豁免）")


def rule_stale(model, report):
    today = date.today()
    for rel in sorted(model.files):
        fm = model.parsed[rel][0]
        if not isinstance(fm, dict):
            continue
        try:
            layer = int(fm.get("layer"))
        except (TypeError, ValueError):
            layer = None
        if layer != 2:
            continue
        name = as_name(rel)
        reasons = []
        if name in model.superseded:
            reasons.append("被 rel:supersedes 指向")
        d = parse_date(fm.get("last-touched"))
        if d and (today - d).days > STALE_DAYS:
            reasons.append("last-touched 超 %d 天" % STALE_DAYS)
        if reasons:
            report(rel, "INFO", "STALE", "疑似陈旧：%s" % "、".join(reasons),
                   "确认是否仍有效；已被取代则标 supersedes/归档（stale 仅提示，不阻断 merge）")


def rule_dup_title(model, report):
    for t, rels in model.titles.items():
        if len(rels) > 1:
            for rel in rels[1:]:
                report(rel, "WARN", "DUP_TITLE", "标题与他卡重复: %s" % t,
                       "合并或改名，避免标题冲突（影响检索与别名归一）")


def rule_dup_alias(model, report):
    for a, rels in model.aliases.items():
        if len(rels) > 1:
            for rel in rels[1:]:
                report(rel, "WARN", "DUP_ALIAS", "别名与他卡冲突: %s" % a,
                       "别名须全局唯一，改其一（别名供 MAP.yaml 归一）")


# ---------------------------------------------------------------- script-volume rules
# Alternative criteria for the heterogeneous node class (G principle). Volumes are
# exempt from knowledge assumptions; these rules are their computable replacement.

def _vol_blocks(txt):
    """All 4+-backtick/tilde fenced blocks in a volume: [(marker, lang, content)]."""
    return [(m.group(1), m.group(2), m.group(3)) for m in VOL_BLOCK_RE.finditer(txt)]


def rule_vol_name_dup(model, report):
    """S5: two volumes sharing a basename make [[name#VERSION]] ambiguous and let a
    shadow overwrite the truth source. Basename must be globally unique among volumes.
    Deterministic ownership: the lexicographically first path keeps the name (the
    model's vol_by_name first-seen-wins on sorted iteration); later paths report."""
    for nm, rels in sorted(model.vol_name_dups.items()):
        if len(rels) > 1:
            owner = min(rels)
            for rel in sorted(rels):
                if rel != owner:
                    report(rel, "ERROR", "VOL_NAME_DUP",
                           "卷文件名与他卷重复: %s（与 %s）" % (nm, owner),
                           "卷 basename 全局唯一（[[名#VERSION]] 按名解析）；改名或合并")


# Knowledge-card frontmatter fields a volume must NOT carry (closed-world whitelist,
# A's "no knowledge-ization boundary" finding): exemption is two-way — volumes are
# exempt from knowledge criteria AND forbidden from wearing knowledge fields.
VOL_FORBIDDEN_FIELDS = ("layer", "k", "v", "e", "edge-policy", "project",
                        "domain", "maturity", "summary", "aliases", "spec-kind",
                        "trigger", "scope", "session-task", "touched", "target")


def rule_vol_knowledge_field(model, report):
    for rel in sorted(model.volumes):
        fm = model.parsed[rel][0]
        if not isinstance(fm, dict):
            continue
        bad = [k for k in VOL_FORBIDDEN_FIELDS if k in fm]
        if bad:
            report(rel, "ERROR", "VOL_KNOWLEDGE_FIELD",
                   "脚本卷携带知识字段: %s" % ", ".join(bad),
                   "卷是非知识异构节点（封闭世界白名单）：移除知识字段，技术字段见 schema script_volume")


def rule_vol_fm(model, report):
    for rel in sorted(model.volumes):
        fm = model.parsed[rel][0]
        if fm is None:
            continue   # FM_MISSING already covers unparseable frontmatter
        for k in VOL_REQUIRED:
            if k not in fm or _empty_required(fm[k]):
                report(rel, "ERROR", "VOL_FM", "脚本卷缺必填技术字段: %s" % k,
                       "按 schema.yaml script_volume.required 补字段 %s" % k)
        _id = fm.get("id")
        if isinstance(_id, str) and _id:
            if not VOL_ID_RE.match(_id):
                report(rel, "ERROR", "VOL_FM", "脚本卷 id 形态非法: %s（应为 vol-{program}）" % _id,
                       "id 用程序名锚定形态 vol-{program}（撞名限定 vol-{dir}-{program} 亦合法，见 schema id_format）")
            stem = as_name(rel).replace("_", "-")
            if not (_id == "vol-" + stem or _id.endswith("-" + stem)):
                report(rel, "ERROR", "VOL_FM",
                       "脚本卷 id 与文件名不同源: %s ↔ %s" % (_id, as_name(rel)),
                       "id↔文件名 stem↔source-of 三者同源（id = vol- + 文件名 stem，下划线转连字符）")
        src = fm.get("source-of")
        if isinstance(src, str) and src:
            if not src.endswith(".py"):
                report(rel, "ERROR", "VOL_FM", "source-of 非 .py 落点: %s" % src,
                       "source-of 是抽取落点=程序调用路径，须以 .py 结尾")
            elif os.path.basename(src)[:-3] != as_name(rel):
                report(rel, "ERROR", "VOL_FM",
                       "source-of 与文件名不同源: %s ↔ %s" % (src, as_name(rel)),
                       "source-of 的脚本 stem 须与卷文件名一致（完整调用身份锚）")
        if fm.get("eol") not in ("crlf", "lf"):
            report(rel, "ERROR", "VOL_FM", "eol 非法: %r" % fm.get("eol"),
                   "eol 取 crlf/lf（代码块行尾约定）")


def rule_vol_fence(model, report):
    for rel in sorted(model.volumes):
        fm, body, txt = model.parsed[rel]
        blocks = _vol_blocks(txt)
        if len(blocks) != 1:
            report(rel, "ERROR", "VOL_FENCE",
                   "脚本卷须恰一个四反引号代码块（实有 %d 个完整块）" % len(blocks),
                   "卷=frontmatter+恰一个 ````python 代码块；检查围栏开闭完整（闭合围栏须顶格）")
            continue
        marker, lang, _ = blocks[0]
        if len(marker) != 4 or marker[0] != "`":
            report(rel, "ERROR", "VOL_FENCE", "代码围栏须为四反引号（实为 %r）" % marker,
                   "用 ````python 开围栏 + ```` 闭围栏（四反引号，容纳代码内三反引号）")
        if lang != "python":
            report(rel, "ERROR", "VOL_FENCE", "代码块语言标记非 python: %r" % lang,
                   "开围栏语言标记须为 python（抽取配方按 ````python 定位）")


def rule_vol_sha256(model, report):
    for rel in sorted(model.volumes):
        fm, body, txt = model.parsed[rel]
        if not isinstance(fm, dict):
            continue
        declared = fm.get("sha256")
        blocks = _vol_blocks(txt)
        if len(blocks) != 1 or not isinstance(declared, str) or not declared:
            continue   # VOL_FENCE / VOL_FM already report these
        content = blocks[0][2]
        # canonical convention (V2): code ends WITHOUT trailing newline; the \r\n
        # before the closing fence is structural. The regex capture includes that
        # separator, so strip exactly one trailing line break before hashing.
        if content.endswith("\r\n"):
            content = content[:-2]
        elif content.endswith("\n"):
            content = content[:-1]
        # v2.7.0 (B6): hash RAW bytes, no eol normalization. vol_extract (the
        # consumer) hashes raw bytes; two validators of one integrity anchor must
        # not disagree. The eol field stays documentary metadata (VOL_FM checks
        # its legality); mixed-EOL code now fails here exactly as extraction does.
        actual = hashlib.sha256(content.encode("utf-8")).hexdigest()
        if actual != declared.strip().lower():
            report(rel, "ERROR", "VOL_SHA256",
                   "代码块 sha256 不符: 声明 %s… 实算 %s…" % (declared[:12], actual[:12]),
                   "改代码块后须同步更新 frontmatter sha256（完整性锚；抽取运行前也会比对）")


def rule_vol_deps(model, report):
    """VOL_DEPS: depends-on is the volume dependency graph (vol ids as primary keys).
    Optional field; when present it must be a list of vol ids that resolve to existing
    volumes. Cycles are rejected here; vol_extract also guards them at extraction time."""
    for rel in sorted(model.volumes):
        fm = model.parsed[rel][0]
        if not isinstance(fm, dict):
            continue
        deps = fm.get("depends-on")
        if deps is None:
            continue
        if not isinstance(deps, list) or any(
                not isinstance(d, str) or not VOL_ID_RE.match(d) for d in deps):
            report(rel, "ERROR", "VOL_DEPS",
                   "depends-on 须为 vol id 列表（vol-{program} 形态）: %r" % deps,
                   "depends-on: [vol-a, vol-b]——引用卷 id（依赖图主键），非文件名/路径")
            continue
        for d in deps:
            if d not in model.vol_by_id:
                report(rel, "ERROR", "VOL_DEPS", "depends-on 引用不存在的卷 id: %s" % d,
                       "确认被依赖卷已建且 id 正确（联邦复制时依赖卷须一并复制）")
        # cycle detection over the resolvable subgraph (DFS, visiting/done sets)
        visiting, done = set(), set()

        def dfs(vid, path):
            if vid in done:
                return None
            if vid in visiting:
                return path[path.index(vid):] + [vid]
            visiting.add(vid)
            r2 = model.vol_by_id.get(vid)
            fm2 = model.parsed[r2][0] if r2 else None
            d2 = fm2.get("depends-on") if isinstance(fm2, dict) else None
            if isinstance(d2, list):
                for nxt in d2:
                    if isinstance(nxt, str) and nxt in model.vol_by_id:
                        cyc = dfs(nxt, path + [vid])
                        if cyc:
                            return cyc
            visiting.discard(vid)
            done.add(vid)
            return None

        own = fm.get("id")
        if isinstance(own, str) and own in model.vol_by_id:
            cyc = dfs(own, [])
            if cyc:
                report(rel, "ERROR", "VOL_DEPS",
                       "depends-on 存在循环依赖: %s" % " → ".join(cyc),
                       "打破循环（抽取器按拓扑序递归抽取，循环不可解）")


def rule_doc_impl_drift(model, report):
    """G20: reconcile current-state claims carried by [[volume#ANCHOR|alias]] links.
    VERSION -> code VERSION const (ERROR). v2.7.0: sole anchor (TCB shrink — RULES/
    SELFTEST removed: decorative counts with forgeable truth channels). Anchor name
    case-insensitive; alias optional; multiple version tokens in one alias → WARN
    (ambiguous claim, S7); prerelease suffix participates in comparison (B8).
    Links declare current-state intent; plain-text mentions are historical and exempt."""
    for rel in sorted(model.files):
        if rel in model.volumes:
            continue   # a volume's own code/fixtures are not doc claims
        txt = model.graph_texts[rel]
        for m in ANCHOR_LINK_RE.finditer(strip_code(txt)):
            target, anchor, alias = m.group(1).strip(), m.group(2).upper(), m.group(3)
            vrel = model.vol_by_name.get(target.split("/")[-1])
            if vrel is None:
                continue   # not a volume target: BROKEN_LINK handles unresolvable names
            if anchor != "VERSION":
                continue   # only VERSION is a reconciled anchor (v2.7.0)
            truth = model.vol_versions.get(vrel)
            if truth is None:
                report(rel, "WARN", "DOC_IMPL_DRIFT",
                       "[[%s#VERSION]] 对账失败：卷代码块未找到 VERSION 常量" % target,
                       "卷代码块须含 VERSION = \"x.y.z\" 常量（版本真值源）")
                continue
            if alias is None or not alias.strip():
                report(rel, "WARN", "DOC_IMPL_DRIFT",
                       "[[%s#VERSION]] 链接未携带版本别名" % target,
                       "写成 [[%s#VERSION|工具 v%s]]，别名写明宣称版本供对账" % (target, truth))
                continue
            matches = CLAIMED_VER_RE.findall(alias)
            if not matches:
                report(rel, "WARN", "DOC_IMPL_DRIFT",
                       "[[%s#VERSION|%s]] 别名未携带版本号" % (target, alias),
                       "别名写明宣称版本（如 doctor v2.6.0），供对账器读取")
                continue
            if len(matches) > 1:
                report(rel, "WARN", "DOC_IMPL_DRIFT",
                       "[[%s#VERSION|%s]] 别名含多个版本号，无法判定宣称" % (target, alias),
                       "一个链接只宣称一个当前版本；历史版本改纯文本叙述")
                continue
            if matches[0] != truth:
                report(rel, "ERROR", "DOC_IMPL_DRIFT",
                       "%s 版本漂移: 文档=%s 实现=%s" % (target, matches[0], truth),
                       "更新文档版本引用为 v%s（真值=卷代码块 VERSION），或确认引用的是历史版本则改纯文本" % truth)


def _schema_tables(text):
    """Extract the contract tables doctor implements from schema.yaml (flat-subset
    line parse, stdlib only): common required keys, type enum, layer enum,
    type-by-layer mapping (from the type comment), script_volume.required keys,
    governance.required keys, overview.required keys (v2.13.0).
    Returns dict or None when the schema is absent/unreadable."""
    if not text:
        return None
    lines = text.splitlines()
    tables = {"common_required": [], "type_enum": [], "layer_enum": [],
              "vol_required": [], "gov_required": [], "ov_required": [],
              "type_by_layer": {}}
    section = None
    for ln in lines:
        if re.match(r"^common:\s*", ln):
            section = "common"; continue
        if re.match(r"^script_volume:\s*", ln):
            section = "vol"; continue
        if re.match(r"^governance:\s*", ln):
            section = "gov"; continue
        if re.match(r"^overview:\s*", ln):
            section = "ov"; continue
        if re.match(r"^[A-Za-z_]", ln) and not ln.startswith(" "):
            section = None
        if section == "common":
            m = re.match(r"^  ([A-Za-z0-9_\-]+):\s*(.*)$", ln)
            if m:
                key, rest = m.group(1), m.group(2)
                if "# 可选" not in rest and "(可选" not in rest:
                    tables["common_required"].append(key)
                if key == "type":
                    me = re.search(r"enum \[([^\]]+)\]", rest)
                    if me:
                        tables["type_enum"] = [x.strip() for x in me.group(1).split(",")]
                    for lm in re.finditer(r"L(\d)→([a-z/]+)", rest):
                        tables["type_by_layer"][int(lm.group(1))] = set(lm.group(2).split("/"))
                if key == "layer":
                    me = re.search(r"enum \[([^\]]+)\]", rest)
                    if me:
                        tables["layer_enum"] = [x.strip() for x in me.group(1).split(",")]
        if section == "vol":
            if re.match(r"^  required:", ln):
                section = "volreq"; continue
        if section == "volreq":
            m = re.match(r"^    ([A-Za-z0-9_\-]+):\s*", ln)
            if m:
                tables["vol_required"].append(m.group(1))
            elif re.match(r"^  [a-z]", ln):
                section = None
        if section == "gov":
            if re.match(r"^  required:", ln):
                section = "govreq"; continue
        if section == "govreq":
            m = re.match(r"^    ([A-Za-z0-9_\-]+):\s*", ln)
            if m:
                tables["gov_required"].append(m.group(1))
            elif re.match(r"^  [a-z]", ln):
                section = None
        if section == "ov":
            if re.match(r"^  required:", ln):
                section = "ovreq"; continue
        if section == "ovreq":
            m = re.match(r"^    ([A-Za-z0-9_\-]+):\s*", ln)
            if m:
                tables["ov_required"].append(m.group(1))
            elif re.match(r"^  [a-z]", ln):
                section = None
    return tables


def rule_contract_drift(model, report):
    """G20 CONTRACT (v2.7.0): reconcile doctor's hardcoded contract constants against
    schema.yaml, the declared authority. Closes A's highest-risk finding (schema↔doctor
    dual authority with no reconciliation path; budgets comment was a live drift instance)."""
    txt = model.files.get(SCHEMA_REL)
    if txt is None:
        # schema.yaml is not .md so load_vault skips it; read from disk when available
        if model.vault_root:
            p = os.path.join(model.vault_root, SCHEMA_REL.replace("/", os.sep))
            if os.path.isfile(p):
                try:
                    with open(p, encoding="utf-8") as fh:
                        txt = fh.read()
                except Exception:
                    txt = None
    tables = _schema_tables(txt)
    if tables is None:
        report(SCHEMA_REL, "WARN", "CONTRACT_DRIFT",
               "CONTRACT 对账失败：schema.yaml 不可读",
               "确认 90-Meta/schema.yaml 存在且可读（契约宣称权威）")
        return
    if tables["common_required"] and set(tables["common_required"]) != set(REQUIRED_COMMON):
        report(SCHEMA_REL, "ERROR", "CONTRACT_DRIFT",
               "CONTRACT 漂移: common 必填字段表 schema=%s doctor=%s"
               % (sorted(tables["common_required"]), sorted(REQUIRED_COMMON)),
               "同步 doctor REQUIRED_COMMON 与 schema common（schema 是宣称权威）")
    if tables["vol_required"] and set(tables["vol_required"]) != set(VOL_REQUIRED):
        report(SCHEMA_REL, "ERROR", "CONTRACT_DRIFT",
               "CONTRACT 漂移: script_volume.required schema=%s doctor=%s"
               % (sorted(tables["vol_required"]), sorted(VOL_REQUIRED)),
               "同步 doctor VOL_REQUIRED 与 schema script_volume.required")
    if tables["gov_required"] and set(tables["gov_required"]) != set(GOV_REQUIRED):
        report(SCHEMA_REL, "ERROR", "CONTRACT_DRIFT",
               "CONTRACT 漂移: governance.required schema=%s doctor=%s"
               % (sorted(tables["gov_required"]), sorted(GOV_REQUIRED)),
               "同步 doctor GOV_REQUIRED 与 schema governance.required")
    if tables["ov_required"] and set(tables["ov_required"]) != set(OV_REQUIRED):
        report(SCHEMA_REL, "ERROR", "CONTRACT_DRIFT",
               "CONTRACT 漂移: overview.required schema=%s doctor=%s"
               % (sorted(tables["ov_required"]), sorted(OV_REQUIRED)),
               "同步 doctor OV_REQUIRED 与 schema overview.required")
    if tables["type_by_layer"]:
        for layer, types in tables["type_by_layer"].items():
            impl = TYPE_BY_LAYER.get(layer, set())
            if types != impl:
                report(SCHEMA_REL, "ERROR", "CONTRACT_DRIFT",
                       "CONTRACT 漂移: L%d type 集合 schema=%s doctor=%s"
                       % (layer, sorted(types), sorted(impl)),
                       "同步 doctor TYPE_BY_LAYER 与 schema common.type 注释的 type↔layer 映射")


# ---------------------------------------------------------------- hub rules (v2.9.0)
# Abstraction axis (DP-5 方案A): hub marking is a structural fact (in-degree >= 2
# strong edges); depth is a semantic judgment belonging to the promotion axis,
# not mixed into hub marking. HUB_* rules reconcile declaration vs evidence (WARN:
# qualification is semantic, Owner judges; mechanical only checks consistency).

def rule_hub_grounds(model, report):
    """hub:true L2 card must have >=2 incoming strong semantic edges
    (implements/grounds/specializes; derives_from excluded = source not instance)."""
    for rel in sorted(model.knowledge):
        fm = model.parsed[rel][0]
        if not isinstance(fm, dict) or not fm.get("hub"):
            continue
        try:
            layer = int(fm.get("layer"))
        except (TypeError, ValueError):
            continue
        if layer != 2:
            report(rel, "WARN", "HUB_GROUNDS",
                   "hub 标记仅限 L2 卡（当前 layer=%s）" % fm.get("layer"),
                   "hub 是 L2 抽象枢纽标记；L1/L3 不标 hub")
            continue
        name = as_name(rel)
        strong = [(r, src) for r, src in model.in_edges.get(name, [])
                  if r in HUB_STRONG_RELS]
        if len(strong) < 2:
            report(rel, "WARN", "HUB_GROUNDS",
                   "hub:true 但强语义入边不足（%d < 2；implements/grounds/specializes）" % len(strong),
                   "入度≥2 是标 hub 的结构证据（DP-5 方案A）；补实例 implements 边或撤 hub 标记")


def rule_hub_decompression(model, report):
    """hub:true card must declare abstracts edges (decompression path), and each
    abstracts edge must have a matching implements inverse (instance->hub).
    Reconciles declaration vs actual instance relations (stronger than prose links)."""
    for rel in sorted(model.knowledge):
        fm = model.parsed[rel][0]
        if not isinstance(fm, dict) or not fm.get("hub"):
            continue
        try:
            layer = int(fm.get("layer"))
        except (TypeError, ValueError):
            continue
        if layer != 2:
            continue   # HUB_GROUNDS already reports non-L2 hub
        name = as_name(rel)
        abs_targets = model.abstracts.get(rel, [])
        if not abs_targets:
            report(rel, "WARN", "HUB_DECOMPRESSION",
                   "hub:true 但无 abstracts 解压路径边",
                   "补 abstracts 边指向实例卡（implements 的逆，解压路径，DP-3 乙）")
            continue
        # each abstracts target must have implements back to this hub
        for tgt in abs_targets:
            inverses = [(r, src) for r, src in model.in_edges.get(name, [])
                        if r == "implements" and as_name(src) == tgt]
            # also check: does the target card implement this hub?
            tgt_rels = [r2 for r2 in model.files if as_name(r2) == tgt]
            has_inverse = False
            for tr in tgt_rels:
                tfm = model.parsed[tr][0]
                if not isinstance(tfm, dict):
                    continue
                tev = tfm.get("e")
                if isinstance(tev, list):
                    for item in tev:
                        if isinstance(item, str) and "implements" in item and name in item:
                            has_inverse = True
            if not has_inverse:
                report(rel, "WARN", "HUB_DECOMPRESSION",
                       "abstracts→[[%s]] 缺对应 implements 反向边" % tgt,
                       "实例卡须有 implements→本枢纽边（互逆对账）；补边或撤 abstracts")


def rule_secondary_type(model, report):
    """secondary-type (DP-4 dual-type transition): legal enum + != type + fission
    plan pointer (存疑节 or changelog mentions 分化). Practice-first, clause later."""
    for rel in sorted(model.knowledge):
        fm = model.parsed[rel][0]
        if not isinstance(fm, dict):
            continue
        st = fm.get("secondary-type")
        if st is None or not str(st).strip():
            continue
        st = str(st).strip()
        typ = fm.get("type")
        if st not in SECONDARY_TYPES:
            report(rel, "ERROR", "SECONDARY_TYPE",
                   "secondary-type 非法枚举: %s" % st,
                   "secondary-type 取知识 type 枚举（concept/claim/procedure/…）")
        if st == typ:
            report(rel, "ERROR", "SECONDARY_TYPE",
                   "secondary-type 与 type 相同: %s" % st,
                   "双 type 过渡须两面不同；相同则删 secondary-type")
        # fission plan pointer: body must mention 分化 (存疑节 or changelog)
        body = model.parsed[rel][1]
        if "分化" not in body:
            report(rel, "WARN", "SECONDARY_TYPE",
                   "secondary-type 缺分化计划指针（正文须注明何时/如何分化）",
                   "在存疑节或 changelog 注明「再结晶-分化」计划（DP-4 第三跃迁范式）")


# ---------------------------------------------------------------- overview rules
# Alternative criteria for the overview category (G principle, volume/governance
# precedent, corr-2026-11-20-89). Overview nodes are human-only README maps, exempt
# from knowledge assumptions; these rules are their computable replacement. Broken
# links stay under BROKEN_LINK (all-rule, ERROR — no OV_LINK: the exemption covers
# the navigation contract, not basic link honesty); UNREACHABLE exempts overview.

def rule_ov_fm(model, report):
    for rel in sorted(model.overview):
        fm = model.parsed[rel][0]
        if fm is None:
            continue   # FM_MISSING already covers unparseable frontmatter
        for k in OV_REQUIRED:
            if k not in fm or _empty_required(fm[k]):
                report(rel, "ERROR", "OV_FM", "overview 缺必填字段: %s" % k,
                       "按 schema.yaml overview.required 补字段 %s" % k)
        typ = fm.get("type")
        if typ is not None and str(typ).strip() and str(typ) not in OV_TYPES:
            report(rel, "ERROR", "OV_FM", "overview type 非法: %r" % typ,
                   "type 取 overview（枚举真值源=schema common.type，OV_FM 自带实现）")
        _id = fm.get("id")
        if isinstance(_id, str) and _id and not OV_ID_RE.match(_id):
            report(rel, "ERROR", "OV_FM",
                   "overview id 形态非法: %s（应为 ov-{yyyy}-{slug}）" % _id,
                   "id 用 ov-{yyyy}-{slug} 形态（如 ov-2026-readme，见 schema overview）")


def rule_ov_knowledge_field(model, report):
    for rel in sorted(model.overview):
        fm = model.parsed[rel][0]
        if not isinstance(fm, dict):
            continue
        bad = [k for k in OV_FORBIDDEN_FIELDS if k in fm]
        if bad:
            report(rel, "ERROR", "OV_KNOWLEDGE_FIELD",
                   "overview 携带知识字段: %s" % ", ".join(bad),
                   "overview 是非知识节点（封闭世界白名单）：移除知识字段，字段见 schema overview（summary/aliases 合法）")


# ---------------------------------------------------------------- governance rules
# Alternative criteria for the governance family (G principle, volume precedent).
# Governance nodes are exempt from knowledge assumptions; these rules are their
# computable replacement. GOV_NAME deferred: Owner ruled zero-rename for the
# numbered governance cards (DP-6), so no computable naming criterion exists yet.

def rule_gov_fm(model, report):
    for rel in sorted(model.governance):
        fm = model.parsed[rel][0]
        if fm is None:
            continue   # FM_MISSING already covers unparseable frontmatter
        for k in GOV_REQUIRED:
            if k not in fm or _empty_required(fm[k]):
                report(rel, "ERROR", "GOV_FM", "治理节点缺必填字段: %s" % k,
                       "按 schema.yaml governance.required 补字段 %s" % k)
        typ = fm.get("type")
        # conditional required fields by type (D0-1 contract)
        cond = {"constraint": ("status", "priority", "scope"),
                "status": ("status", "scope"),
                "reference": ("status",)}.get(typ, ())
        for k in cond:
            if k not in fm or _empty_required(fm[k]):
                report(rel, "ERROR", "GOV_FM",
                       "治理节点 type=%s 缺条件必填字段: %s" % (typ, k),
                       "按 schema.yaml governance.conditional 补字段 %s" % k)
        _id = fm.get("id")
        if isinstance(_id, str) and _id:
            if typ == "status":
                if _id != "gov-status":
                    report(rel, "ERROR", "GOV_FM",
                           "status 单例 id 须为 gov-status: %s" % _id,
                           "type:status 全库单例，id 固定 gov-status（P-5）")
            elif not GOV_ID_RE.match(_id):
                report(rel, "ERROR", "GOV_FM",
                       "治理节点 id 形态非法: %s（应为 gov-{yyyy}-{slug}）" % _id,
                       "id 用 gov-{yyyy}-{slug} 形态（status 单例例外 gov-status）")
        pri = fm.get("priority")
        if pri is not None and str(pri).strip() and str(pri) not in ("P0", "P1", "P2"):
            report(rel, "ERROR", "GOV_FM", "priority 非法: %r" % pri,
                   "priority 取 P0/P1/P2（P0=违反即阻断任务）")
        scp = fm.get("scope")
        if scp is not None and str(scp).strip() and str(scp) != "workspace-agent":
            report(rel, "ERROR", "GOV_FM", "scope 非法: %r" % scp,
                   "scope 取 workspace-agent（治理族作用域=本工作区 Agent 行为）")


def rule_gov_status(model, report):
    for rel in sorted(model.governance):
        fm = model.parsed[rel][0]
        if not isinstance(fm, dict):
            continue
        st = fm.get("status")
        typ = fm.get("type")
        if st is None or _empty_required(st):
            continue   # GOV_FM reports missing conditional status
        legal = ("ACTIVE",) if typ == "status" else ("ACTIVE", "SUPERSEDED")
        if str(st) not in legal:
            report(rel, "ERROR", "GOV_STATUS", "status 非法: %r（type=%s 合法值 %s）"
                   % (st, typ, "/".join(legal)),
                   "status 取 ACTIVE/SUPERSEDED（type:status 单例仅 ACTIVE）")
        if str(st) == "SUPERSEDED":
            sup = fm.get("supersedes")
            if not isinstance(sup, str) or not sup.strip():
                report(rel, "ERROR", "GOV_STATUS",
                       "SUPERSEDED 缺 supersedes 指针",
                       "补 supersedes: 取代者指针（[[新节点]] 或 gov id）")
            else:
                m = re.search(r"\[\[([^\]\|#]+)", sup)
                tgt = m.group(1).strip() if m else sup.strip()
                name = tgt.split("/")[-1]
                if name not in model.resolvable and tgt not in model.ids:
                    report(rel, "ERROR", "GOV_STATUS",
                           "supersedes 指针目标不存在: %s" % sup,
                           "确认取代者节点已建（wikilink 或 gov id 可解析）")


def rule_gov_entry(model, report):
    """Final form (batch 4, 00-ENTRY internalized): ERROR. Anchor = 05-Governance/00-INDEX
    nav page (00-ENTRY links it; the nav page IS the anchor and is exempt). A node
    reachable from ROOT but absent from the nav page still fires: the governance family
    must be discoverable through its own entry, not by accident."""
    for rel in sorted(model.governance):
        if as_name(rel) == "00-INDEX":
            continue
        if as_name(rel) not in model.gov_reachable:
            report(rel, "ERROR", "GOV_ENTRY",
                   "治理节点经入口链不可达（锚=05-Governance/00-INDEX）",
                   "在 05-Governance/00-INDEX 导航页建立指向本节点的链接（治理族须经自身入口可发现）")


def rule_gov_knowledge_field(model, report):
    for rel in sorted(model.governance):
        fm = model.parsed[rel][0]
        if not isinstance(fm, dict):
            continue
        bad = [k for k in GOV_FORBIDDEN_FIELDS if k in fm]
        if bad:
            report(rel, "ERROR", "GOV_KNOWLEDGE_FIELD",
                   "治理节点携带知识字段: %s" % ", ".join(bad),
                   "治理族是非知识节点（封闭世界白名单）：移除知识字段，治理字段见 schema governance")


def rule_gov_singleton(model, report):
    """type:status is a vault-wide singleton (gov-status). >1 fires ERROR on the
    later paths (sorted ownership, ID_DUP precedent). 0 is legal during batches
    1-2 (STATUS not yet internalized); batch 3 adds the presence requirement."""
    sts = sorted(rel for rel in model.governance
                 if isinstance(model.parsed[rel][0], dict)
                 and model.parsed[rel][0].get("type") == "status")
    if len(sts) == 0 and model.vault_root:
        report("05-Governance/", "ERROR", "GOV_SINGLETON",
               "type:status 单例缺失（批3起恰1）",
               "建 05-Governance/STATUS.md（id:gov-status, type:status）")
    if len(sts) > 1:
        for rel in sts[1:]:
            report(rel, "ERROR", "GOV_SINGLETON",
                   "type:status 单例冲突（与 %s）" % sts[0],
                   "全库恰一个 type:status 节点（gov-status）；合并或改 type")


def rule_status_claim(model, report):
    """STATUS_CLAIM: 6 mechanical reconciliations against disk truth.
    Only fires when vault_root is set (real scan, not self-test fixtures).
    Checks: doctor version, scripts count, instruments count, corr count,
    maturity declaration log exists, AUDITABLE declaration exists."""
    if not model.vault_root:
        return
    sts = [rel for rel in model.governance
           if isinstance(model.parsed[rel][0], dict)
           and model.parsed[rel][0].get("type") == "status"]
    if not sts:
        return
    rel = sts[0]
    fm, body, txt = model.parsed[rel]
    root = model.vault_root

    # 1. doctor version: STATUS mentions v{VERSION}
    ver_pat = "v" + VERSION
    if ver_pat not in body:
        report(rel, "ERROR", "STATUS_CLAIM",
               "doctor 版本宣称缺失或漂移（应含 %s）" % ver_pat,
               "更新 STATUS doctor 版本为代码真值 %s" % VERSION)

    # 2. scripts/ .py count: only files WITHOUT a corresponding volume are "resident"
    #    (extracted copies are transient; doctor itself runs from one)
    scripts_dir = os.path.join(root, "90-Meta", "scripts")
    if os.path.isdir(scripts_dir):
        vol_names = set(model.vol_by_name.keys())  # e.g. {"entry_doctor", "append_log", ...}
        resident = [f for f in os.listdir(scripts_dir)
                    if f.endswith(".py") and f[:-3] not in vol_names]
        if resident:
            report(rel, "ERROR", "STATUS_CLAIM",
                   "scripts/ 常驻 .py = %d（无对应卷：%s）" % (len(resident), ", ".join(resident)),
                   "清理无卷对应的 .py，或建卷")

    # 3. instruments/ .py count == 8
    inst_dir = os.path.join(root, "..", "Audit", "History",
                            "TTL-Round-3-Superseded", "instruments")
    inst_dir = os.path.normpath(inst_dir)
    if os.path.isdir(inst_dir):
        inst_count = len([f for f in os.listdir(inst_dir) if f.endswith(".py")])
        if inst_count != 8:
            report(rel, "ERROR", "STATUS_CLAIM",
                   "instruments/ .py = %d（应 8）" % inst_count,
                   "核验冻结仪器完整性")

    # 4. corrections count: STATUS mentions N 件; compare to disk
    corr_dir = os.path.join(root, "04-Logs", "corrections")
    if os.path.isdir(corr_dir):
        corr_count = len([f for f in os.listdir(corr_dir) if f.endswith(".md")])
        # look for "Corrections**: N 件" pattern in body
        m = re.search(r"Corrections\*?\*?:?\s*(\d+)\s*件", body)
        if m and int(m.group(1)) != corr_count:
            report(rel, "ERROR", "STATUS_CLAIM",
                   "corrections 宣称 %s 件 != 磁盘 %d 件" % (m.group(1), corr_count),
                   "更新 STATUS corrections 计数为磁盘实测")

    # 5. maturity declaration log exists (log-2026-11-14-01)
    decl_log = os.path.join(root, "04-Logs", "logs", "log-2026-11-14-01.md")
    if not os.path.isfile(decl_log):
        report(rel, "ERROR", "STATUS_CLAIM",
               "maturity 宣告 log 缺失（log-2026-11-14-01）",
               "核验宣告留痕完整性")

    # 6. AUDITABLE declaration in V2-EXECUTION-RECORD
    v2_rec = os.path.normpath(os.path.join(
        root, "..", "Audit", "Current",
        "TTL-Core-Hypothesis-Spec-NON-AUTHORITY",
        "ROUND-3C-V2-EXECUTION-RECORD.md"))
    if os.path.isfile(v2_rec):
        v2_txt = open(v2_rec, encoding="utf-8").read()
        if "AUDITABLE" not in v2_txt:
            report(rel, "ERROR", "STATUS_CLAIM",
                   "V2-EXECUTION-RECORD 缺 AUDITABLE 宣告段",
                   "核验终审宣告完整性")


def rule_status_size(model, report):
    """STATUS_SIZE: body >200 lines -> WARN (prevent design diary recurrence)."""
    sts = [rel for rel in model.governance
           if isinstance(model.parsed[rel][0], dict)
           and model.parsed[rel][0].get("type") == "status"]
    for rel in sts:
        fm, body, txt = model.parsed[rel]
        lines = body.count("\n") + 1
        if lines > 200:
            report(rel, "WARN", "STATUS_SIZE",
                   "STATUS 正文 %d 行 > 200（防 design diary 复发）" % lines,
                   "瘦身：历史叙述下沉 L1，只留当前态+指针")


RULES = [
    Rule("FM_MISSING", "file", rule_fm_missing),
    Rule("FIELD_MISSING", "file", rule_field_missing, applies=("knowledge",)),
    Rule("ID_DUP", "file", rule_id_dup),
    Rule("ID_NAMESPACE", "file", rule_id_namespace, since="2.3.0"),
    Rule("LAYER_BAD", "file", rule_layer_bad, applies=("knowledge",)),
    Rule("TYPE_LAYER", "file", rule_type_layer, applies=("knowledge",)),
    Rule("SPEC_KIND", "file", rule_spec_kind, applies=("knowledge",)),
    Rule("TRIGGER_FM", "file", rule_trigger_fm, since="2.14.0", applies=("knowledge",)),
    Rule("SUMMARY_LONG", "file", rule_summary_long, level="WARN", applies=("knowledge",)),
    Rule("ATLAS_EMPTY", "file", rule_atlas_empty, applies=("knowledge",)),
    Rule("ATLAS_GROUP", "file", rule_atlas_group, level="WARN", since="2.15.0", applies=("knowledge",)),
    Rule("EDGE_ORPHAN", "file", rule_edge_orphan, applies=("knowledge",)),
    Rule("LOG_SECTION", "file", rule_log_section, applies=("knowledge",)),
    Rule("CORR_SECTION", "file", rule_corr_section, applies=("knowledge",)),
    Rule("CORR_TARGET", "file", rule_corr_target, applies=("knowledge",)),
    Rule("SPEC_CHANGELOG", "file", rule_spec_changelog, applies=("knowledge",)),
    Rule("BROKEN_LINK", "link", rule_broken_link),
    Rule("UNREACHABLE", "graph", rule_unreachable, level="WARN"),
    Rule("STALE", "graph", rule_stale, level="INFO", applies=("knowledge",)),
    Rule("DUP_TITLE", "graph", rule_dup_title, level="WARN"),
    Rule("DUP_ALIAS", "graph", rule_dup_alias, level="WARN"),
    Rule("VOL_FM", "file", rule_vol_fm, since="2.4.0", applies=("volume",)),
    Rule("VOL_FENCE", "file", rule_vol_fence, since="2.4.0", applies=("volume",)),
    Rule("VOL_SHA256", "file", rule_vol_sha256, since="2.4.0", applies=("volume",)),
    Rule("VOL_DEPS", "cross", rule_vol_deps, since="2.5.0"),
    Rule("VOL_NAME_DUP", "file", rule_vol_name_dup, since="2.7.0", applies=("volume",)),
    Rule("VOL_KNOWLEDGE_FIELD", "file", rule_vol_knowledge_field, since="2.7.0", applies=("volume",)),
    Rule("DOC_IMPL_DRIFT", "cross", rule_doc_impl_drift, level="ERROR", since="2.4.0"),
    Rule("CONTRACT_DRIFT", "cross", rule_contract_drift, level="ERROR", since="2.7.0"),
    Rule("GOV_FM", "file", rule_gov_fm, since="2.8.0", applies=("governance",)),
    Rule("GOV_STATUS", "file", rule_gov_status, since="2.8.0", applies=("governance",)),
    Rule("GOV_ENTRY", "graph", rule_gov_entry, level="ERROR", since="2.8.0", applies=("governance",)),
    Rule("GOV_KNOWLEDGE_FIELD", "file", rule_gov_knowledge_field, since="2.8.0", applies=("governance",)),
    Rule("GOV_SINGLETON", "graph", rule_gov_singleton, since="2.8.0", applies=("governance",)),
    Rule("OV_FM", "file", rule_ov_fm, since="2.13.0", applies=("overview",)),
    Rule("OV_KNOWLEDGE_FIELD", "file", rule_ov_knowledge_field, since="2.13.0", applies=("overview",)),
    Rule("STATUS_CLAIM", "cross", rule_status_claim, since="2.10.0", applies=("governance",)),
    Rule("STATUS_SIZE", "file", rule_status_size, level="WARN", since="2.10.0", applies=("governance",)),
    Rule("HUB_GROUNDS", "graph", rule_hub_grounds, level="WARN", since="2.9.0", applies=("knowledge",)),
    Rule("HUB_DECOMPRESSION", "graph", rule_hub_decompression, level="WARN", since="2.9.0", applies=("knowledge",)),
    Rule("SECONDARY_TYPE", "file", rule_secondary_type, since="2.9.0", applies=("knowledge",)),
]


# ---------------------------------------------------------------- engine

def analyze(files, extra_resolvable=None, vault_root=None):
    """files: {relpath(posix): text} -> (issues, counts, broken). issue = dict(file, level, rule, message, fix).
    Builds a read-only VaultModel, then dispatches each registered rule. Rules are stateless;
    the engine collects diagnostics. Output ordering is normalized by output() (file, level, rule)."""
    model = VaultModel(files, extra_resolvable, vault_root=vault_root)
    issues = []
    broken = set()

    def report(rel, level, rule, msg, fix):
        issues.append({"file": rel, "level": level, "rule": rule,
                       "message": msg, "fix": fix})
        if rule == "BROKEN_LINK":
            m = re.search(r"\[\[([^\]]+)\]\]", msg)
            if m:
                broken.add(m.group(1))

    for rule in RULES:
        if rule.applies == ("all",):
            rule.fn(model, report)
        else:
            # Central applicability dispatch (RC4): the rule sees a category-filtered
            # view; per-rule guards are no longer the enforcement point.
            allowed = set()
            if "knowledge" in rule.applies:
                allowed |= model.knowledge
            if "volume" in rule.applies:
                allowed |= model.volumes
            if "governance" in rule.applies:
                allowed |= model.governance
            if "overview" in rule.applies:
                allowed |= model.overview
            sub = VaultModel.__new__(VaultModel)
            sub.__dict__.update(model.__dict__)
            sub.files = {rel: txt for rel, txt in model.files.items() if rel in allowed}
            rule.fn(sub, report)

    counts = {"error": 0, "warn": 0, "info": 0}
    for it in issues:
        counts[it["level"].lower()] += 1
    return issues, counts, broken


# ---------------------------------------------------------------- CLI cross-check

def vault_display_name(root):
    """Obsidian identifies a vault by its folder name; the CLI's global `vault=<name>`
    option selects on that name."""
    if not root:
        return ""
    return os.path.basename(os.path.normpath(os.path.abspath(root)))


# `obsidian unresolved` answers for whichever vault is targeted, so it is only usable
# when the vault is pinned. These replies mean the pin did not land on a usable vault
# (wrong vault / vault not open / command not registered there) -- they are error
# notices, never an empty link list.
CLI_UNPINNED_RE = re.compile(
    r'^(?:vault not found|error:|command\s+"[^"]*"\s+not found)', re.I | re.M)


def _cli_output_unusable(text):
    """True when the CLI reply is an error / wrong-vault notice rather than a link list."""
    return bool(CLI_UNPINNED_RE.search(text or ""))


def _cli_file_found(text):
    """Did `obsidian file path=<rel>` resolve the path (info block) rather than error out?"""
    s = (text or "").strip()
    return bool(s) and not _cli_output_unusable(s)


def _cli_probe_files(files):
    """Pick a few exact vault-relative paths the offline scan actually saw, shallow first
    (so a TTL vault's 00-ROOT.md is tried before deep cards)."""
    if not files:
        return []
    keys = sorted(files, key=lambda p: (p.count("/"), p))
    return keys[:3]


def _cli_run(args, timeout):
    """Run an obsidian CLI command, decoding its output as UTF-8 explicitly.

    The CLI emits UTF-8 (this is a Chinese vault). text=True alone decodes via
    the Windows locale (GBK/cp936), which raises UnicodeDecodeError on Chinese
    replies and leaves stdout=None (D10). encoding="utf-8" matches append_log.py.
    """
    return subprocess.run(args, capture_output=True, text=True,
                          encoding="utf-8", timeout=timeout)


def _cli_sees_path(name, rel):
    """Does the pinned CLI resolve this exact vault-relative path?"""
    try:
        r = _cli_run(["obsidian", "vault=%s" % name, "file", "path=%s" % rel], 15)
    except Exception:
        return False
    return _cli_file_found(((r.stdout or "") + (r.stderr or "")))


def use_cli_crosscheck(root, probe_files=()):
    """Best-effort `obsidian unresolved`, pinned to *this* vault and identity-checked.

    The command answers for the vault selected by the CLI's global `vault=<name>`
    option (default: the focused window), so without the pin a `--vault --use-cli`
    run reconciles one vault against another and invents CLI_RECONCILE divergences
    that do not exist.

    The pin alone is not proof: `vault=<name>` selects by folder name, so two vaults
    sharing a name -- or a focused window on a different vault -- pass a version
    check and still answer for the wrong vault. So the pin is confirmed by resolving
    a path the offline scan just saw; only then is the unresolved list trusted.
    Anything else degrades to offline with a note naming the cause.
    """
    name = vault_display_name(root)
    if not name:
        return None, "无法从 vault 根推导 vault 名，降级离线"
    scope = ["vault=%s" % name]
    try:
        ver = _cli_run(["obsidian"] + scope + ["version"], 15)
    except Exception as e:
        return None, "obsidian CLI 不可用（%s），降级离线" % type(e).__name__
    if ver.returncode != 0:
        return None, "obsidian CLI 返回非零，降级离线"
    vreply = ((ver.stdout or "") + (ver.stderr or "")).strip()
    if _cli_output_unusable(vreply):
        return None, ("obsidian 未以 vault「%s」打开（%s），无法对账该 vault，降级离线"
                      % (name, vreply.splitlines()[0][:60] if vreply else "空回复"))
    cands = _cli_probe_files(probe_files)
    if not cands:
        return None, "没有可用于确认 vault 身份的文件，无法确认 CLI 指向本 vault，降级离线"
    if not any(_cli_sees_path(name, rel) for rel in cands):
        return None, ("obsidian 的 vault「%s」看不到本 vault 的文件（如 %s），"
                      "疑似同名 vault 冲突或 CLI 指向别的窗口，降级离线"
                      % (name, cands[0]))
    try:
        un = _cli_run(["obsidian"] + scope + ["unresolved"], 30)
    except Exception as e:
        return None, "obsidian unresolved 调用失败（%s），降级离线" % type(e).__name__
    reply = ((un.stdout or "") + (un.stderr or "")).strip()
    if un.returncode != 0 or _cli_output_unusable(reply):
        return None, ("obsidian 无法在 vault「%s」执行 unresolved（%s；"
                      "常见原因：同名 vault 冲突或该处未启用该命令），降级离线"
                      % (name, reply.splitlines()[0][:60] if reply else
                         "rc=%d" % un.returncode))
    return un.stdout.strip(), None


NON_WIKILINK_EXT_RE = re.compile(
    r"\.(md|markdown|yaml|yml|png|jpe?g|gif|svg|pdf|canvas|json|webp)$", re.I)

# obsidian unresolved prints a literal sentinel string when there are no broken links; it is not a link target and must be filtered out (else a false WARN is raised)
CLI_EMPTY_RE = re.compile(r"^no unresolved links? found\.?$", re.I)

# --use-cli reconciles against the App's in-memory index, which may lag the working tree (P0-b: spec first, not a hard gate)
INDEX_FRESHNESS_NOTE = ("对账基于 App 内存索引，可能滞后于工作树；"
                        "换分支/大改后请先重载 vault 再复跑（规约见 SPEC-GOVERN §分支切换）")


def _cli_comparable(raw):
    """Is a CLI unresolved item a bare note name that the offline wikilink scan can cover?
    Directory links (trailing slash), path-style (contains slash), and non-md targets with
    an extension -> not comparable (the offline scan does not cover them)."""
    s = raw.strip()
    if not s or s.endswith("/") or "/" in s or "\\" in s:
        return False
    return not NON_WIKILINK_EXT_RE.search(s)


def reconcile_cli(cli_text, offline_broken):
    """Auto-reconcile the raw obsidian unresolved output against the offline broken set.
    Return (notes, warns): notes = classification summary dict; warns = [(level, rule, msg, fix)]
    for genuine divergences only. Semantic gap: offline scans only [[wikilink]] while the CLI also
    covers dir / markdown links / embeds, so classify by comparability, not a naive set diff."""
    cli_raw = [ln.strip() for ln in (cli_text or "").splitlines()
               if ln.strip() and not CLI_EMPTY_RE.match(ln.strip())]
    cli_wl = {x for x in cli_raw if _cli_comparable(x)}
    cli_oos = {x for x in cli_raw if not _cli_comparable(x)}
    cli_all = {x.rstrip("/") for x in cli_raw}
    off = {x.rstrip("/") for x in offline_broken}
    cli_wl_n = {x.rstrip("/") for x in cli_wl}
    agree = off & cli_wl_n
    offline_only = off - cli_all
    cli_only_wl = cli_wl_n - off
    warns = []
    for t in sorted(offline_only):
        warns.append(("WARN", "CLI_RECONCILE",
                      "离线判断链 [[%s]] 但 obsidian unresolved 未列" % t,
                      "可能离线误报(该名实为别名/已解析)，或 App 索引滞后于工作树；重载 vault 后复跑 --use-cli 复核"))
    for t in sorted(cli_only_wl):
        warns.append(("WARN", "CLI_RECONCILE",
                      "obsidian unresolved 列 [[%s]] 未解析，但离线未标记" % t,
                      "可能离线正则漏判，或该 wikilink 目标确缺；建目标笔记或修正链接后复跑"))
    notes = {
        "cli_unresolved": sorted(cli_all),
        "agree_broken": sorted(agree),
        "offline_only": sorted(offline_only),
        "cli_only_wikilink": sorted(cli_only_wl),
        "cli_only_out_of_scope": sorted(cli_oos),
        "index_freshness_note": INDEX_FRESHNESS_NOTE,
    }
    return notes, warns


# ---------------------------------------------------------------- self-test

def _vol_fix(name, code="print('hi')\n", **kw):
    """Build a well-formed script-volume fixture (frontmatter + one ````python block).
    sha256 computed over the code with the declared eol, so a clean fixture passes VOL_*."""
    eol = kw.get("eol", "crlf")
    raw = code.replace("\r\n", "\n").rstrip("\n")   # canonical: no trailing newline
    raw = raw.replace("\n", "\r\n") if eol == "crlf" else raw
    base = {"id": "vol-" + name.replace("_", "-"), "type": "script-volume",
            "created": "2026-11-16", "last-touched": "2026-11-16",
            "source-of": "90-Meta/scripts/%s.py" % name,
            "fence": '"````"', "eol": eol,
            "sha256": hashlib.sha256(raw.encode("utf-8")).hexdigest()}
    base.update(kw)
    fm = "\n".join("%s: %s" % (k, v) for k, v in base.items())
    body = code.replace("\r\n", "\n").rstrip("\n")   # canonical: no trailing newline
    body = body.replace("\n", "\r\n") if eol == "crlf" else body
    return "---\n%s\n---\n````python\n%s\n````\n" % (fm, body)


def _gov_fix(slug, typ="reference", **kw):
    """Build a well-formed governance fixture (flat frontmatter, no knowledge fields)."""
    base = {"id": "gov-2026-" + slug, "type": typ, "status": "ACTIVE",
            "created": "2026-11-16", "last-touched": "2026-11-16"}
    if typ == "constraint":
        base.update({"priority": "P1", "scope": "workspace-agent"})
    if typ == "status":
        base.update({"id": "gov-status", "scope": "workspace-agent"})
    base.update(kw)
    fm = "\n".join("%s: %s" % (k, v) for k, v in base.items())
    return "---\n%s\n---\n# %s\n" % (fm, slug)


def _ov_fix(slug, **kw):
    """Build a well-formed overview fixture (human-only README map, no knowledge fields)."""
    base = {"id": "ov-2026-" + slug, "type": "overview",
            "created": "2026-11-20", "last-touched": "2026-11-20",
            "summary": "全景地图"}
    base.update(kw)
    fm = "\n".join("%s: %s" % (k, v) for k, v in base.items())
    return "---\n%s\n---\n# %s\n" % (fm, slug)


def _fix(rel, **kw):
    base = {"id": "x-" + rel, "layer": 2, "type": "concept", "project": "memory-lib",
            "domain": "[test]", "maturity": "seed", "created": "2026-09-27",
            "last-touched": "2026-09-27", "summary": "test"}
    base.update({k: v for k, v in kw.items()})
    fm = "\n".join("%s: %s" % (k, base[k]) for k in REQUIRED_COMMON
                   if k in base) if False else "\n".join(
        "%s: %s" % (k, base[k]) for k in
        ["id", "layer", "type", "project", "domain", "maturity",
         "created", "last-touched", "summary"])
    extra = "\n".join("%s: %s" % (k, v) for k, v in kw.items()
                      if k not in base)
    return "---\n%s\n%s\n---\n" % (fm, extra) if extra else "---\n%s\n---\n" % fm


def self_test():
    """Built-in destructive fixtures (in-memory, no disk writes): verify each rule can fire."""
    files = {
        "00-ROOT.md": ("---\nid: \"spec-root-000\"\nlayer: 3\ntype: spec\nspec-kind: read\n"
                       "project: memory-lib\ndomain: [test]\nmaturity: growing\ncreated: 2026-09-27\n"
                       "last-touched: 2026-09-27\nsummary: root\n---\n# ROOT\n"
                       "- [good](02-Cards/good.md)\n- [atlas](03-Atlas/)\n"),
        "02-Cards/good.md": ("---\nid: \"c-good\"\nlayer: 2\ntype: concept\nproject: memory-lib\n"
                             "domain: [test]\nmaturity: seed\ncreated: 2026-09-27\nlast-touched: 2026-09-27\n"
                             "summary: good\ne:\n  - { target: \"[[good2]]\", rel: co_occurs, weight: 0.5 }\n"
                             "---\n# good\n到 [[good2]] 的边\n"),
        "02-Cards/good2.md": ("---\nid: \"c-good2\"\nlayer: 2\ntype: concept\nproject: memory-lib\n"
                              "domain: [test]\nmaturity: seed\ncreated: 2026-09-27\nlast-touched: 2026-09-27\n"
                              "summary: g2\ne:\n  - { target: \"[[good]]\", rel: co_occurs, weight: 0.5 }\n"
                              "---\n# good2\n"),
        "03-Atlas/SH-test.md": ("---\nid: \"s-atlas\"\nlayer: 2\ntype: summary\nproject: memory-lib\n"
                                "domain: [test]\nmaturity: seed\ncreated: 2026-09-27\nlast-touched: 2026-09-27\n"
                                "summary: atlas\nedge-policy: exempt\n---\n# SH-test\n"
                                "<!-- atlas:auto-start -->\n- [[good]]\n<!-- atlas:auto-end -->\n"),
        # -- deliberately broken --
        "02-Cards/broken-orphan.md": _fix("c-orphan") + "# orphan\n无边\n",
        "02-Cards/broken-typelayer.md": _fix("c-tl", layer=2, type="spec") + "# tl\n",
        "02-Cards/broken-link.md": (_fix("c-bl") +
                                    "e:\n  - { target: \"[[good]]\", rel: co_occurs, weight: 0.5 }\n"
                                    "# bl\n指向 [[NoSuchNoteXYZ]]\n").replace(
                                        "summary: test\n---", "summary: test\n---"),
        "02-Cards/broken-missfield.md": ("---\nid: \"c-mf\"\nlayer: 2\ntype: concept\nproject: memory-lib\n"
                                         "maturity: seed\ncreated: 2026-09-27\nlast-touched: 2026-09-27\n"
                                         "summary: mf\ne:\n  - { target: \"[[good]]\", rel: co_occurs, weight: 0.5 }\n"
                                         "---\n# mf\n"),
        "02-Cards/broken-dupid.md": ("---\nid: \"c-good\"\nlayer: 2\ntype: concept\nproject: memory-lib\n"
                                     "domain: [test]\nmaturity: seed\ncreated: 2026-09-27\nlast-touched: 2026-09-27\n"
                                     "summary: dup\ne:\n  - { target: \"[[good]]\", rel: co_occurs, weight: 0.5 }\n"
                                     "---\n# dupid\n"),
        "04-Logs/broken-log.md": ("---\nid: \"log-2026-09-27-99\"\nlayer: 1\ntype: log\nproject: memory-lib\n"
                                  "domain: [test]\nmaturity: evergreen\ncreated: 2026-09-27\nlast-touched: 2026-09-27\n"
                                  "summary: lg\n---\n# log\n只有一段\n"),
        "03-Atlas/broken-atlas.md": ("---\nid: \"s-badatlas\"\nlayer: 2\ntype: summary\nproject: memory-lib\n"
                                     "domain: [test]\nmaturity: seed\ncreated: 2026-09-27\nlast-touched: 2026-09-27\n"
                                     "summary: bad\nedge-policy: exempt\n---\n# bad atlas\n"
                                     "<!-- atlas:auto-start -->\n<!-- atlas:auto-end -->\n"),
        "02-Cards/broken-unreach.md": ("---\nid: \"c-unreach\"\nlayer: 2\ntype: concept\nproject: memory-lib\n"
                                       "domain: [test]\nmaturity: seed\ncreated: 2026-09-27\nlast-touched: 2026-09-27\n"
                                       "summary: ur\ne:\n  - { target: \"[[good]]\", rel: co_occurs, weight: 0.5 }\n"
                                       "---\n# unreach\n没人链我\n"),
        "02-Cards/broken-stale.md": ("---\nid: \"c-stale\"\nlayer: 2\ntype: concept\nproject: memory-lib\n"
                                     "domain: [test]\nmaturity: seed\ncreated: 2000-01-01\nlast-touched: 2000-01-01\n"
                                     "summary: st\ne:\n  - { target: \"[[good]]\", rel: co_occurs, weight: 0.5 }\n"
                                     "---\n# stale\n"),
        # unparseable frontmatter: quoted scalar with trailing content after the closing quote
        "02-Cards/broken-fm.md": ("---\nid: \"c-badfm\"\nlayer: 2\ntype: concept\nproject: memory-lib\n"
                                  "domain: [test]\nmaturity: seed\ncreated: 2026-09-27\nlast-touched: 2026-09-27\n"
                                  "summary: \"引号\"尾巴\n"
                                  "e:\n  - { target: \"[[good]]\", rel: co_occurs, weight: 0.5 }\n"
                                  "---\n# badfm\n"),
    }
    # broken-link needs a real edge + a broken link; build separately to avoid _fix interference
    files["02-Cards/broken-link.md"] = (
        "---\nid: \"c-bl\"\nlayer: 2\ntype: concept\nproject: memory-lib\ndomain: [test]\n"
        "maturity: seed\ncreated: 2026-09-27\nlast-touched: 2026-09-27\nsummary: bl\n"
        "e:\n  - { target: \"[[good]]\", rel: co_occurs, weight: 0.5 }\n---\n"
        "# bl\n指向 [[NoSuchNoteXYZ]]\n")
    # v0.7 correction fixtures: missing sections + missing target
    files["04-Logs/corrections/broken-corr.md"] = (
        "---\nid: \"corr-2026-09-27-90\"\nlayer: 1\ntype: correction\nproject: memory-lib\n"
        "domain: [test]\nmaturity: evergreen\ncreated: 2026-09-27\nlast-touched: 2026-09-27\n"
        "summary: cr\n---\n# corr\n只有一段\n")
    # v0.7 spec embedding a changelog body (should be sunk)
    files["01-Specs/broken-spec-cl.md"] = (
        "---\nid: \"spec-cl\"\nlayer: 3\ntype: spec\nspec-kind: read\nproject: memory-lib\n"
        "domain: [test]\nmaturity: growing\ncreated: 2026-09-27\nlast-touched: 2026-09-27\n"
        "summary: sc\ntrigger: always\n---\n# spec\n正文\n\n## changelog\n\n- v0.1: x\n")

    # id-namespace fixtures (federation): a declared sub-vault where a new card wrongly
    # continues the upstream id segment, while an upstream copy (provenance: copied) and
    # a properly prefixed new card must both pass.
    # P1b: fixtures for the 5 rules previously lacking destructive coverage
    files["02-Cards/broken-layer.md"] = _fix("c-layer", layer="abc") + "# layer\n"
    files["01-Specs/broken-speckind.md"] = _fix("c-sk", layer=3, type="spec") + "# sk\n"
    files["02-Cards/broken-longsum.md"] = _fix("c-ls", summary="x" * 130) + "# ls\n"
    files["02-Cards/broken-duptitle-a.md"] = _fix("c-dta") + "# 重复标题样本\n"
    files["02-Cards/broken-duptitle-b.md"] = _fix("c-dtb") + "# 重复标题样本\n"
    files["02-Cards/broken-dupalias-a.md"] = _fix("c-daa")[:-4] + "aliases:\n  - dupaliasx\n---\n# daa\n"
    files["02-Cards/broken-dupalias-b.md"] = _fix("c-dab")[:-4] + "aliases:\n  - dupaliasx\n---\n# dab\n"
    ns_files = {
        "00-ROOT.md": ("---\nid: \"spec-root-000\"\nlayer: 3\ntype: spec\nspec-kind: read\n"
                       "project: demo\ndomain: [test]\nmaturity: growing\ncreated: 2026-09-27\n"
                       "last-touched: 2026-09-27\nsummary: root\nid-namespace: demo\n---\n# ROOT\n"
                       "- [a](02-Cards/ns-bad.md)\n- [b](02-Cards/ns-copy.md)\n- [c](02-Cards/ns-good.md)\n"),
        "02-Cards/ns-bad.md": (
            "---\nid: \"c-2026-00099\"\nlayer: 2\ntype: concept\nproject: demo\ndomain: [test]\n"
            "maturity: seed\ncreated: 2026-09-27\nlast-touched: 2026-09-27\nsummary: bad\n"
            "e:\n  - { target: \"[[ns-good]]\", rel: co_occurs, weight: 0.5 }\n---\n# ns-bad\n"),
        "02-Cards/ns-copy.md": (
            "---\nid: \"c-2026-00020\"\nlayer: 2\ntype: concept\nproject: demo\ndomain: [test]\n"
            "maturity: seed\ncreated: 2026-09-27\nlast-touched: 2026-09-27\nsummary: copy\n"
            "provenance: copied\nsource: \"[[上游库/ns-origin]]\"\n---\n# ns-copy\n上游复制卡\n"),
        "02-Cards/ns-copy-nosrc.md": (
            "---\nid: \"c-2026-00021\"\nlayer: 2\ntype: concept\nproject: demo\ndomain: [test]\n"
            "maturity: seed\ncreated: 2026-09-27\nlast-touched: 2026-09-27\nsummary: copy2\n"
            "provenance: copied\n---\n# ns-copy-nosrc\n缺来源佐证的复制卡\n"),
        "02-Cards/ns-good.md": (
            "---\nid: \"c-demo-00001\"\nlayer: 2\ntype: concept\nproject: demo\ndomain: [test]\n"
            "maturity: seed\ncreated: 2026-09-27\nlast-touched: 2026-09-27\nsummary: good\n"
            "---\n# ns-good\n本库新卡\n"),
    }
    ns_issues, _, _ = analyze(ns_files)
    ns_bad = [it for it in ns_issues if it["rule"] == "ID_NAMESPACE" and it["level"] == "ERROR"]
    ns_warn = [it for it in ns_issues if it["rule"] == "ID_NAMESPACE" and it["level"] == "WARN"]
    ns_ok = (len(ns_bad) == 1 and ns_bad[0]["file"] == "02-Cards/ns-bad.md"
             and len(ns_warn) == 1 and ns_warn[0]["file"] == "02-Cards/ns-copy-nosrc.md")

    # script-volume fixtures (V2): a clean volume must pass ALL rules (no knowledge
    # fields required), and each VOL_* rule needs a destructive counterpart.
    vol_files = {
        "00-ROOT.md": ("---\nid: \"spec-root-000\"\nlayer: 3\ntype: spec\nspec-kind: read\n"
                       "project: memory-lib\ndomain: [test]\nmaturity: growing\ncreated: 2026-11-16\n"
                       "last-touched: 2026-11-16\nsummary: root\n---\n# ROOT\n"
                       "- [vol](90-Meta/script-volumes/vol_good.md)\n"),
        "90-Meta/script-volumes/vol_good.md": _vol_fix("vol_good"),
        # VOL_FM: missing required tech field (sha256) + id not same-source as filename
        "90-Meta/script-volumes/vol_badfm.md": _vol_fix("vol_badfm", id="vol-other-name").replace(
            "sha256: %s\n" % hashlib.sha256(b"print('hi')").hexdigest(), ""),
        # VOL_FENCE: two complete blocks (must be exactly one)
        "90-Meta/script-volumes/vol_badfence.md": _vol_fix("vol_badfence") + "````python\nprint(2)\n````\n",
        # VOL_SHA256: declared hash does not match the code block
        "90-Meta/script-volumes/vol_badsha.md": _vol_fix("vol_badsha", sha256="0" * 64),
        # VOL_KNOWLEDGE_FIELD: volume wearing a knowledge field (closed-world whitelist)
        "90-Meta/script-volumes/vol_knowfield.md": _vol_fix("vol_knowfield", layer=2),
        # VOL_NAME_DUP: two volumes sharing a basename (shadow collision, S5).
        # sub/ sorts first and owns the name; the root-level twin reports. Both
        # carry the same-source id — ID_DUP firing too is correct (same id twice);
        # the clean-volume assertion only requires no KNOWLEDGE rule on vol_good.
        "90-Meta/script-volumes/sub/vol_good.md": _vol_fix("good"),
    }
    vol_issues, _, _ = analyze(vol_files)
    vol_good_issues = [it for it in vol_issues if it["file"] == "90-Meta/script-volumes/vol_good.md"
                       and it["rule"] not in ("VOL_NAME_DUP", "ID_DUP")]
    vol_expect = {"90-Meta/script-volumes/vol_badfm.md": "VOL_FM",
                  "90-Meta/script-volumes/vol_badfence.md": "VOL_FENCE",
                  "90-Meta/script-volumes/vol_badsha.md": "VOL_SHA256",
                  "90-Meta/script-volumes/vol_knowfield.md": "VOL_KNOWLEDGE_FIELD",
                  "90-Meta/script-volumes/vol_good.md": "VOL_NAME_DUP"}
    vol_ok = not vol_good_issues
    for f, r in vol_expect.items():
        hits = [it for it in vol_issues if it["file"] == f and it["rule"] == r]
        vol_ok = vol_ok and bool(hits)
    # knowledge rules must NOT fire on the clean volume (exemption works), even
    # when the volume wears a knowledge field (central dispatch, not per-rule guards)
    vol_ok = vol_ok and not any(it["rule"] in ("FIELD_MISSING", "LAYER_BAD", "TYPE_LAYER",
                                               "EDGE_ORPHAN", "STALE", "ATLAS_EMPTY")
                                for it in vol_issues if it["file"].endswith("vol_good.md"))
    vol_ok = vol_ok and not any(it["rule"] in ("EDGE_ORPHAN", "STALE", "ATLAS_EMPTY", "LAYER_BAD")
                                for it in vol_issues if it["file"].endswith("vol_knowfield.md"))

    # VOL_DEPS fixtures: good chain passes; missing target / malformed list / cycle fire.
    vd_files = {
        "00-ROOT.md": ("---\nid: \"spec-root-000\"\nlayer: 3\ntype: spec\nspec-kind: read\n"
                       "project: memory-lib\ndomain: [test]\nmaturity: growing\ncreated: 2026-11-16\n"
                       "last-touched: 2026-11-16\nsummary: root\n---\n# ROOT\n"
                       "- [a](90-Meta/script-volumes/vol_base.md)\n"
                       "- [b](90-Meta/script-volumes/vol_mid.md)\n"
                       "- [c](90-Meta/script-volumes/vol_missdep.md)\n"
                       "- [d](90-Meta/script-volumes/vol_baddep.md)\n"
                       "- [e](90-Meta/script-volumes/vol_cyc_a.md)\n"
                       "- [f](90-Meta/script-volumes/vol_cyc_b.md)\n"),
        "90-Meta/script-volumes/vol_base.md": _vol_fix("base"),
        "90-Meta/script-volumes/vol_mid.md": _vol_fix("mid", **{"depends-on": "[vol-base]"}),
        "90-Meta/script-volumes/vol_missdep.md": _vol_fix("missdep", **{"depends-on": "[vol-ghost]"}),
        "90-Meta/script-volumes/vol_baddep.md": _vol_fix("baddep", **{"depends-on": "vol-base"}),
        "90-Meta/script-volumes/vol_cyc_a.md": _vol_fix("cyc_a", **{"depends-on": "[vol-cyc-b]"}),
        "90-Meta/script-volumes/vol_cyc_b.md": _vol_fix("cyc_b", **{"depends-on": "[vol-cyc-a]"}),
    }
    vd_issues, _, _ = analyze(vd_files)
    vd_hits = [it for it in vd_issues if it["rule"] == "VOL_DEPS"]
    vd_ok = (not any(it["file"].endswith(("vol_base.md", "vol_mid.md")) for it in vd_hits)
             and any(it["file"].endswith("vol_missdep.md") and "vol-ghost" in it["message"] for it in vd_hits)
             and any(it["file"].endswith("vol_baddep.md") for it in vd_hits)
             and any("循环" in it["message"] for it in vd_hits))

    # G20 DOC_IMPL_DRIFT fixtures (v2.7.0): VERSION is the sole anchor.
    # vol_tool code carries VERSION = 1.2.3. Doc claims: correct / drifted /
    # lowercase anchor / no-alias / multi-version alias / prerelease suffix /
    # path-form link; plain-text history stays exempt.
    g20_code = 'VERSION = "1.2.3"\n'
    g20_files = {
        "00-ROOT.md": ("---\nid: \"spec-root-000\"\nlayer: 3\ntype: spec\nspec-kind: read\n"
                       "project: memory-lib\ndomain: [test]\nmaturity: growing\ncreated: 2026-11-16\n"
                       "last-touched: 2026-11-16\nsummary: root\n---\n# ROOT\n"
                       "- [vol](90-Meta/script-volumes/vol_tool.md)\n"
                       "- [doc](02-Cards/g20-doc.md)\n"),
        "90-Meta/script-volumes/vol_tool.md": _vol_fix("vol_tool", code=g20_code),
        "02-Cards/g20-doc.md": (
            "---\nid: \"c-g20\"\nlayer: 2\ntype: concept\nproject: memory-lib\ndomain: [test]\n"
            "maturity: seed\ncreated: 2026-11-16\nlast-touched: 2026-11-16\nsummary: g20\n"
            "e:\n  - { target: \"[[vol_tool]]\", rel: co_occurs, weight: 0.5 }\n---\n# g20\n"
            "正确 [[vol_tool#VERSION|tool v1.2.3]]；"
            "漂移 [[vol_tool#VERSION|tool v9.9.9]]；"
            "小写锚点漂移 [[vol_tool#version|tool v8.8.8]]；"
            "无别名 [[vol_tool#VERSION]]；"
            "多版本别名 [[vol_tool#VERSION|v1.2.3 已升级至 v9.9.9]]；"
            "预发布漂移 [[vol_tool#VERSION|tool v1.2.3-beta]]；"
            "路径形态漂移 [[90-Meta/script-volumes/vol_tool#VERSION|tool v7.7.7]]；"
            "历史纯文本 tool v0.0.1 不纳入对账。\n"),
    }
    g20_issues, _, _ = analyze(g20_files)
    g20_hits = [it for it in g20_issues if it["rule"] == "DOC_IMPL_DRIFT"]
    g20_ver = [it for it in g20_hits if "版本漂移" in it["message"]]
    g20_noalias = [it for it in g20_hits if "未携带版本别名" in it["message"]]
    g20_multi = [it for it in g20_hits if "多个版本号" in it["message"]]
    g20_drifted = {it["message"].split("文档=")[1].split(" ")[0]
                   for it in g20_ver if "文档=" in it["message"]}
    g20_path_fp = [it for it in g20_issues if it["rule"] == "BROKEN_LINK"
                   and "90-Meta/script-volumes/vol_tool" in it["message"]]
    g20_ok = (len(g20_ver) == 4 and all(it["level"] == "ERROR" for it in g20_ver)
              and g20_drifted == {"9.9.9", "8.8.8", "1.2.3-beta", "7.7.7"}
              and len(g20_noalias) == 1 and g20_noalias[0]["level"] == "WARN"
              and len(g20_multi) == 1 and g20_multi[0]["level"] == "WARN"
              and not g20_path_fp)

    # v2.7.0 parser-layer fixtures: phantom fence must NOT hide a live broken link (S2);
    # unclosed fence at EOF stays code (FP3); whitespace/null/empty-list required fields
    # fire FIELD_MISSING (S3); duplicate YAML keys void the whole frontmatter (S4);
    # block scalar outside the flat subset voids it too (fail-loud); broken markdown
    # .md link fires BROKEN_LINK (S8).
    files["02-Cards/broken-phantomfence.md"] = (
        _fix("c-pf") + "# pf\n\n开头```a` 说明。\n\n真断链 [[RT_Ghost_PF]] 是活链接。\n\n"
        "结尾````b 文字。\n")
    files["02-Cards/broken-eoffence.md"] = (
        _fix("c-eof") + "# eof\n\n````python\nprint('[[NotALink_EOF]]')\n")
    files["02-Cards/broken-wsfield.md"] = _fix("c-ws", summary='"   "') + "# ws\n"
    files["02-Cards/broken-nullfield.md"] = _fix("c-nf", summary="null") + "# nf\n"
    files["02-Cards/broken-dupkey.md"] = (
        _fix("c-dk").replace("summary: test", "summary: 首值\nsummary: 次值") + "# dk\n")
    files["02-Cards/broken-blockscalar.md"] = (
        _fix("c-bs").replace("summary: test", "summary: >-\n  折叠标量在子集外") + "# bs\n")
    files["02-Cards/broken-mdlink.md"] = (
        _fix("c-ml") + "# ml\n指向 [ghost](RT_Ghost_MD.md) 的 markdown 断链\n")
    p2_issues, _, _ = analyze(files)
    pf_hits = [it for it in p2_issues if it["file"].endswith("broken-phantomfence.md")
               and it["rule"] == "BROKEN_LINK" and "RT_Ghost_PF" in it["message"]]
    eof_hits = [it for it in p2_issues if it["file"].endswith("broken-eoffence.md")
                and it["rule"] == "BROKEN_LINK"]
    ws_hits = [it for it in p2_issues if it["file"].endswith("broken-wsfield.md")
               and it["rule"] == "FIELD_MISSING"]
    null_hits = [it for it in p2_issues if it["file"].endswith("broken-nullfield.md")
                 and it["rule"] == "FIELD_MISSING"]
    dk_hits = [it for it in p2_issues if it["file"].endswith("broken-dupkey.md")
               and it["rule"] == "FM_MISSING"]
    bs_hits = [it for it in p2_issues if it["file"].endswith("broken-blockscalar.md")
               and it["rule"] == "FM_MISSING"]
    ml_hits = [it for it in p2_issues if it["file"].endswith("broken-mdlink.md")
               and it["rule"] == "BROKEN_LINK" and "RT_Ghost_MD" in it["message"]]
    parser_ok = (bool(pf_hits) and not eof_hits and bool(ws_hits) and bool(null_hits)
                 and bool(dk_hits) and bool(bs_hits) and bool(ml_hits))

    # CONTRACT_DRIFT: schema tables vs doctor constants. Positive: real schema text
    # matching the constants passes; negative: a mutated required list fires ERROR.
    schema_good = (
        "common:\n"
        "  id:          string\n"
        "  layer:       enum [1, 2, 3]\n"
        "  type:        enum [concept, claim, procedure, observation, entity, summary, pitfall, spec, log, report, correction, feedback, script-volume, constraint, reference, status, overview]   # L1→log/report/correction；L2→concept/claim/procedure/observation/entity/summary；L3→spec/pitfall/feedback\n"
        "  project:     string\n"
        "  domain:      list[string]\n"
        "  maturity:    enum [seed, growing, evergreen, archived]\n"
        "  created:     date\n"
        "  last-touched: date\n"
        "  summary:     string\n"
        "  risk:        enum [normal, high] # 可选(默认normal)\n"
        "script_volume:\n"
        "  required:\n"
        "    id:            string\n"
        "    type:          script-volume\n"
        "    created:       date\n"
        "    last-touched:  date\n"
        "    source-of:     string\n"
        "    fence:         string\n"
        "    eol:           enum [crlf, lf]\n"
        "    sha256:        string\n"
        "  optional:\n"
        "    depends-on:    list[string]\n"
        "governance:\n"
        "  required:\n"
        "    id:       string\n"
        "    type:     enum [constraint, reference, status]\n"
        "    created:  date\n"
        "    last-touched: date\n"
        "  conditional:\n"
        "    constraint: { status: enum [ACTIVE, SUPERSEDED], priority: enum [P0, P1, P2], scope: enum [workspace-agent] }\n"
        "overview:\n"
        "  required:\n"
        "    id:            string\n"
        "    type:          enum [overview]\n"
        "    created:       date\n"
        "    last-touched:  date\n"
        "    summary:       string\n"
        "  optional:\n"
        "    aliases:       list[string]\n")
    # hub fixtures (v2.9.0, corr-2026-11-16-58): clean hub with 2 implements in-edges
    # + abstracts decompression passes; hub without enough in-edges / without abstracts /
    # abstracts without inverse each fire WARN; secondary-type illegal/same/no-plan fire.
    hub_files = {
        "00-ROOT.md": ("---\nid: \"spec-root-000\"\nlayer: 3\ntype: spec\nspec-kind: read\n"
                       "project: memory-lib\ndomain: [test]\nmaturity: growing\ncreated: 2026-11-16\n"
                       "last-touched: 2026-11-16\nsummary: root\n---\n# ROOT\n"
                       "- [h](02-Cards/hub-good.md)\n"),
        "02-Cards/hub-good.md": (
            "---\nid: c-hub-good\nlayer: 2\ntype: concept\nproject: memory-lib\ndomain: [test]\n"
            "maturity: seed\ncreated: 2026-11-16\nlast-touched: 2026-11-16\nsummary: hub\n"
            "hub: true\n"
            "e:\n  - { target: \"[[inst-a]]\", rel: abstracts, weight: 0.7 }\n"
            "  - { target: \"[[inst-b]]\", rel: abstracts, weight: 0.7 }\n---\n# hub-good\n"),
        "02-Cards/inst-a.md": (
            "---\nid: c-inst-a\nlayer: 2\ntype: procedure\nproject: memory-lib\ndomain: [test]\n"
            "maturity: seed\ncreated: 2026-11-16\nlast-touched: 2026-11-16\nsummary: a\n"
            "e:\n  - { target: \"[[hub-good]]\", rel: implements, weight: 0.7 }\n---\n# inst-a\n"),
        "02-Cards/inst-b.md": (
            "---\nid: c-inst-b\nlayer: 2\ntype: procedure\nproject: memory-lib\ndomain: [test]\n"
            "maturity: seed\ncreated: 2026-11-16\nlast-touched: 2026-11-16\nsummary: b\n"
            "e:\n  - { target: \"[[hub-good]]\", rel: implements, weight: 0.7 }\n---\n# inst-b\n"),
        # HUB_GROUNDS: hub with only 1 strong in-edge
        "02-Cards/hub-lowdeg.md": (
            "---\nid: c-hub-low\nlayer: 2\ntype: concept\nproject: memory-lib\ndomain: [test]\n"
            "maturity: seed\ncreated: 2026-11-16\nlast-touched: 2026-11-16\nsummary: low\n"
            "hub: true\n"
            "e:\n  - { target: \"[[inst-a]]\", rel: abstracts, weight: 0.7 }\n---\n# hub-lowdeg\n"),
        # HUB_DECOMPRESSION: hub with 2 in-edges but no abstracts
        "02-Cards/hub-noabs.md": (
            "---\nid: c-hub-noabs\nlayer: 2\ntype: concept\nproject: memory-lib\ndomain: [test]\n"
            "maturity: seed\ncreated: 2026-11-16\nlast-touched: 2026-11-16\nsummary: noabs\n"
            "hub: true\n"
            "e:\n  - { target: \"[[inst-a]]\", rel: co_occurs, weight: 0.5 }\n---\n# hub-noabs\n"),
        "02-Cards/inst-c.md": (
            "---\nid: c-inst-c\nlayer: 2\ntype: procedure\nproject: memory-lib\ndomain: [test]\n"
            "maturity: seed\ncreated: 2026-11-16\nlast-touched: 2026-11-16\nsummary: c\n"
            "e:\n  - { target: \"[[hub-noabs]]\", rel: implements, weight: 0.7 }\n---\n# inst-c\n"),
        "02-Cards/inst-d.md": (
            "---\nid: c-inst-d\nlayer: 2\ntype: procedure\nproject: memory-lib\ndomain: [test]\n"
            "maturity: seed\ncreated: 2026-11-16\nlast-touched: 2026-11-16\nsummary: d\n"
            "e:\n  - { target: \"[[hub-noabs]]\", rel: implements, weight: 0.7 }\n---\n# inst-d\n"),
        # HUB_DECOMPRESSION: abstracts without implements inverse
        "02-Cards/hub-badinv.md": (
            "---\nid: c-hub-badinv\nlayer: 2\ntype: concept\nproject: memory-lib\ndomain: [test]\n"
            "maturity: seed\ncreated: 2026-11-16\nlast-touched: 2026-11-16\nsummary: badinv\n"
            "hub: true\n"
            "e:\n  - { target: \"[[inst-a]]\", rel: abstracts, weight: 0.7 }\n"
            "  - { target: \"[[inst-noinv]]\", rel: abstracts, weight: 0.7 }\n---\n# hub-badinv\n"),
        "02-Cards/inst-noinv.md": (
            "---\nid: c-inst-noinv\nlayer: 2\ntype: procedure\nproject: memory-lib\ndomain: [test]\n"
            "maturity: seed\ncreated: 2026-11-16\nlast-touched: 2026-11-16\nsummary: noinv\n"
            "e:\n  - { target: \"[[hub-good]]\", rel: co_occurs, weight: 0.5 }\n---\n# inst-noinv\n"),
        "02-Cards/inst-e.md": (
            "---\nid: c-inst-e\nlayer: 2\ntype: procedure\nproject: memory-lib\ndomain: [test]\n"
            "maturity: seed\ncreated: 2026-11-16\nlast-touched: 2026-11-16\nsummary: e\n"
            "e:\n  - { target: \"[[hub-badinv]]\", rel: implements, weight: 0.7 }\n---\n# inst-e\n"),
        "02-Cards/inst-f.md": (
            "---\nid: c-inst-f\nlayer: 2\ntype: procedure\nproject: memory-lib\ndomain: [test]\n"
            "maturity: seed\ncreated: 2026-11-16\nlast-touched: 2026-11-16\nsummary: f\n"
            "e:\n  - { target: \"[[hub-badinv]]\", rel: implements, weight: 0.7 }\n---\n# inst-f\n"),
        # SECONDARY_TYPE: illegal enum / same as type / no fission plan
        "02-Cards/sec-illegal.md": (
            "---\nid: c-sec-ill\nlayer: 2\ntype: concept\nproject: memory-lib\ndomain: [test]\n"
            "maturity: seed\ncreated: 2026-11-16\nlast-touched: 2026-11-16\nsummary: ill\n"
            "secondary-type: governance\n"
            "e:\n  - { target: \"[[hub-good]]\", rel: co_occurs, weight: 0.5 }\n---\n# sec-illegal\n分化计划\n"),
        "02-Cards/sec-same.md": (
            "---\nid: c-sec-same\nlayer: 2\ntype: concept\nproject: memory-lib\ndomain: [test]\n"
            "maturity: seed\ncreated: 2026-11-16\nlast-touched: 2026-11-16\nsummary: same\n"
            "secondary-type: concept\n"
            "e:\n  - { target: \"[[hub-good]]\", rel: co_occurs, weight: 0.5 }\n---\n# sec-same\n分化计划\n"),
        "02-Cards/sec-noplan.md": (
            "---\nid: c-sec-noplan\nlayer: 2\ntype: procedure\nproject: memory-lib\ndomain: [test]\n"
            "maturity: seed\ncreated: 2026-11-16\nlast-touched: 2026-11-16\nsummary: noplan\n"
            "secondary-type: concept\n"
            "e:\n  - { target: \"[[hub-good]]\", rel: co_occurs, weight: 0.5 }\n---\n# sec-noplan\n无计划\n"),
    }
    hub_issues, _, _ = analyze(hub_files)
    hub_clean = [it for it in hub_issues if it["file"] == "02-Cards/hub-good.md"
                 and it["rule"] in ("HUB_GROUNDS", "HUB_DECOMPRESSION")]
    hub_ok = not hub_clean
    hub_ok = hub_ok and any(it["file"] == "02-Cards/hub-lowdeg.md" and it["rule"] == "HUB_GROUNDS"
                            for it in hub_issues)
    hub_ok = hub_ok and any(it["file"] == "02-Cards/hub-noabs.md" and it["rule"] == "HUB_DECOMPRESSION"
                            and "无 abstracts" in it["message"] for it in hub_issues)
    hub_ok = hub_ok and any(it["file"] == "02-Cards/hub-badinv.md" and it["rule"] == "HUB_DECOMPRESSION"
                            and "inst-noinv" in it["message"] for it in hub_issues)
    hub_ok = hub_ok and any(it["file"] == "02-Cards/sec-illegal.md" and it["rule"] == "SECONDARY_TYPE"
                            and it["level"] == "ERROR" for it in hub_issues)
    hub_ok = hub_ok and any(it["file"] == "02-Cards/sec-same.md" and it["rule"] == "SECONDARY_TYPE"
                            and it["level"] == "ERROR" for it in hub_issues)
    hub_ok = hub_ok and any(it["file"] == "02-Cards/sec-noplan.md" and it["rule"] == "SECONDARY_TYPE"
                            and it["level"] == "WARN" for it in hub_issues)
    # clean instances must not fire hub rules (they are not hubs)
    hub_ok = hub_ok and not any(it["rule"] in ("HUB_GROUNDS", "HUB_DECOMPRESSION")
                                for it in hub_issues if it["file"].startswith("02-Cards/inst-"))

    # boolean scalar fixtures (v2.12.0, corr-2026-11-18-65): unquoted false must
    # parse to False (hub rules skip it); unquoted true to True; quoted "false"
    # stays a truthy string (YAML semantics). Red-before-green: pre-fix parser
    # returned the string "false" (truthy) and misjudged the card as a hub.
    bool_files = dict(hub_files)
    bool_files["02-Cards/bool-false.md"] = (
        "---\nid: c-bool-f\nlayer: 2\ntype: concept\nproject: memory-lib\ndomain: [test]\n"
        "maturity: seed\ncreated: 2026-11-18\nlast-touched: 2026-11-18\nsummary: bf\n"
        "hub: false\n"
        "e:\n  - { target: \"[[hub-good]]\", rel: co_occurs, weight: 0.5 }\n---\n# bool-false\n")
    bool_files["02-Cards/bool-qfalse.md"] = (
        "---\nid: c-bool-qf\nlayer: 2\ntype: concept\nproject: memory-lib\ndomain: [test]\n"
        "maturity: seed\ncreated: 2026-11-18\nlast-touched: 2026-11-18\nsummary: bqf\n"
        "hub: \"false\"\n"
        "e:\n  - { target: \"[[hub-good]]\", rel: co_occurs, weight: 0.5 }\n---\n# bool-qfalse\n")
    bool_issues, _, _ = analyze(bool_files)
    bfm, _ = parse_fm(bool_files["02-Cards/bool-false.md"])
    qfm, _ = parse_fm(bool_files["02-Cards/bool-qfalse.md"])
    hfm, _ = parse_fm(bool_files["02-Cards/hub-good.md"])
    bool_ok = bfm.get("hub") is False and hfm.get("hub") is True
    bool_ok = bool_ok and qfm.get("hub") == "false"
    bool_ok = bool_ok and not any(it["file"] == "02-Cards/bool-false.md"
                                  and it["rule"] in ("HUB_GROUNDS", "HUB_DECOMPRESSION")
                                  for it in bool_issues)
    bool_ok = bool_ok and any(it["file"] == "02-Cards/bool-qfalse.md"
                              and it["rule"] == "HUB_GROUNDS" for it in bool_issues)

    # governance fixtures (v2.8.0, corr-2026-11-16-57): clean reference/constraint
    # pass ALL rules (knowledge exemption two-way); each GOV_* rule needs a
    # destructive counterpart; ID_NAMESPACE exempts gov-* in a declared sub-vault.
    gov_files = {
        "00-ROOT.md": ("---\nid: \"spec-root-000\"\nlayer: 3\ntype: spec\nspec-kind: read\n"
                       "project: memory-lib\ndomain: [test]\nmaturity: growing\ncreated: 2026-11-16\n"
                       "last-touched: 2026-11-16\nsummary: root\n---\n# ROOT\n"
                       "- [idx](05-Governance/00-INDEX.md)\n"),
        "05-Governance/00-INDEX.md": _gov_fix("gov-index") + (
            "- [[gov-good]]\n- [[gov-constraint]]\n- [[gov-badfm]]\n- [[gov-badid]]\n"
            "- [[gov-badcond]]\n- [[gov-badstatus]]\n- [[gov-nosup]]\n- [[gov-badsup]]\n"
            "- [[gov-knowfield]]\n- [[gov-status-a]]\n- [[gov-status-b]]\n"),
        "05-Governance/gov-good.md": _gov_fix("good"),
        "05-Governance/gov-constraint.md": _gov_fix("constraint", typ="constraint"),
        # GOV_FM: missing required (last-touched) + illegal id form + constraint missing priority
        "05-Governance/gov-badfm.md": _gov_fix("badfm").replace("last-touched: 2026-11-16\n", ""),
        "05-Governance/gov-badid.md": _gov_fix("badid", id="gov-terms"),
        "05-Governance/gov-badcond.md": _gov_fix("badcond", typ="constraint").replace("priority: P1\n", ""),
        # GOV_STATUS: illegal enum + SUPERSEDED without pointer + dangling pointer
        "05-Governance/gov-badstatus.md": _gov_fix("badstatus", status="RETIRED"),
        "05-Governance/gov-nosup.md": _gov_fix("nosup", status="SUPERSEDED"),
        "05-Governance/gov-badsup.md": _gov_fix("badsup", status="SUPERSEDED",
                                                supersedes="gov-2026-ghost"),
        # GOV_KNOWLEDGE_FIELD: governance node wearing knowledge fields
        "05-Governance/gov-knowfield.md": _gov_fix("knowfield", layer=2, maturity="seed"),
        # GOV_ENTRY: reachable from ROOT but NOT via the 00-INDEX nav anchor -> ERROR
        "05-Governance/gov-unreach.md": _gov_fix("unreach"),
        # GOV_SINGLETON: two status nodes (first owns, second reports)
        "05-Governance/gov-status-a.md": _gov_fix("status-a", typ="status"),
        "05-Governance/gov-status-b.md": _gov_fix("status-b", typ="status"),
    }
    gov_files["00-ROOT.md"] = gov_files["00-ROOT.md"].replace(
        "- [idx](05-Governance/00-INDEX.md)\n",
        "- [idx](05-Governance/00-INDEX.md)\n- [ur](05-Governance/gov-unreach.md)\n")
    gov_issues, _, _ = analyze(gov_files)
    gov_clean = [it for it in gov_issues
                 if it["file"] in ("05-Governance/gov-good.md", "05-Governance/gov-constraint.md",
                                   "05-Governance/00-INDEX.md")]
    gov_expect = {"05-Governance/gov-badfm.md": "GOV_FM",
                  "05-Governance/gov-badid.md": "GOV_FM",
                  "05-Governance/gov-badcond.md": "GOV_FM",
                  "05-Governance/gov-badstatus.md": "GOV_STATUS",
                  "05-Governance/gov-nosup.md": "GOV_STATUS",
                  "05-Governance/gov-badsup.md": "GOV_STATUS",
                  "05-Governance/gov-knowfield.md": "GOV_KNOWLEDGE_FIELD",
                  "05-Governance/gov-status-b.md": "GOV_SINGLETON"}
    gov_ok = not gov_clean
    for f, r in gov_expect.items():
        gov_ok = gov_ok and any(it["file"] == f and it["rule"] == r for it in gov_issues)
    # GOV_ENTRY: nav-unreachable fires ERROR (batch 4); clean nodes reachable via nav pass
    gov_entry_hits = [it for it in gov_issues if it["rule"] == "GOV_ENTRY"]
    gov_ok = gov_ok and len(gov_entry_hits) == 1 \
        and gov_entry_hits[0]["file"] == "05-Governance/gov-unreach.md" \
        and gov_entry_hits[0]["level"] == "ERROR"
    # knowledge rules must NOT fire on governance nodes (central dispatch, two-way)
    gov_ok = gov_ok and not any(it["rule"] in ("FIELD_MISSING", "LAYER_BAD", "TYPE_LAYER",
                                               "EDGE_ORPHAN", "STALE", "ATLAS_EMPTY", "SPEC_KIND")
                                for it in gov_issues if it["file"].startswith("05-Governance/"))
    # ID_NAMESPACE federation boundary: declared sub-vault, gov-* node exempt (no ERROR)
    gov_ns_files = {
        "00-ROOT.md": ("---\nid: \"spec-root-000\"\nlayer: 3\ntype: spec\nspec-kind: read\n"
                       "project: demo\ndomain: [test]\nmaturity: growing\ncreated: 2026-11-16\n"
                       "last-touched: 2026-11-16\nsummary: root\nid-namespace: demo\n---\n# ROOT\n"
                       "- [g](05-Governance/gov-ns.md)\n"),
        "05-Governance/gov-ns.md": _gov_fix("ns"),
    }
    gov_ns_issues, _, _ = analyze(gov_ns_files)
    gov_ok = gov_ok and not any(it["rule"] == "ID_NAMESPACE" for it in gov_ns_issues)

    # overview fixtures (v2.13.0, corr-2026-11-20-89): clean README passes ALL rules
    # (knowledge exemption two-way + UNREACHABLE explicit exemption even when nobody
    # links it); OV_FM/OV_KNOWLEDGE_FIELD need destructive counterparts; BROKEN_LINK
    # stays ERROR on overview (no OV_LINK); ID_NAMESPACE exempts ov-* in a sub-vault.
    ov_files = {
        "00-ROOT.md": ("---\nid: \"spec-root-000\"\nlayer: 3\ntype: spec\nspec-kind: read\n"
                       "project: memory-lib\ndomain: [test]\nmaturity: growing\ncreated: 2026-11-20\n"
                       "last-touched: 2026-11-20\nsummary: root\n---\n# ROOT\n"
                       "- [r](03-Atlas/README.md)\n"),
        # 干净 overview：aliases 合法；故意无人链（UNREACHABLE 豁免须兜住）
        "03-Atlas/README.md": _ov_fix("readme", aliases="[项目地图]"),
        # OV_FM: 缺必填（last-touched）
        "03-Atlas/ov-badfm.md": _ov_fix("badfm").replace("last-touched: 2026-11-20\n", ""),
        # OV_FM: id 形态非法（无年份段）
        "03-Atlas/ov-badid.md": _ov_fix("badid", id="ov-readme"),
        # OV_KNOWLEDGE_FIELD: overview 穿知识字段（summary 必填合法，不在此列）
        "03-Atlas/ov-knowfield.md": _ov_fix("knowfield", layer=2, maturity="seed"),
        # BROKEN_LINK 不豁免：overview 断链仍 ERROR（all 规则，不另立 OV_LINK）
        "03-Atlas/ov-badlink.md": _ov_fix("badlink") + "指向 [[OV_Ghost_XYZ]]\n",
    }
    ov_issues, _, _ = analyze(ov_files)
    ov_clean = [it for it in ov_issues if it["file"] == "03-Atlas/README.md"]
    ov_expect = {"03-Atlas/ov-badfm.md": "OV_FM",
                 "03-Atlas/ov-badid.md": "OV_FM",
                 "03-Atlas/ov-knowfield.md": "OV_KNOWLEDGE_FIELD"}
    ov_ok = not ov_clean
    for f, r in ov_expect.items():
        ov_ok = ov_ok and any(it["file"] == f and it["rule"] == r for it in ov_issues)
    # UNREACHABLE 显式豁免：干净 overview 无人链也不报 WARN（P1 硬伤回归）
    ov_ok = ov_ok and not any(it["rule"] == "UNREACHABLE"
                              and it["file"] == "03-Atlas/README.md" for it in ov_issues)
    # BROKEN_LINK 不豁免：overview 断链仍 ERROR
    ov_ok = ov_ok and any(it["file"] == "03-Atlas/ov-badlink.md" and it["rule"] == "BROKEN_LINK"
                          and it["level"] == "ERROR" and "OV_Ghost_XYZ" in it["message"]
                          for it in ov_issues)
    # 知识规则不泄漏到 overview（中央分派，双向豁免）
    ov_ok = ov_ok and not any(it["rule"] in ("FIELD_MISSING", "LAYER_BAD", "TYPE_LAYER",
                                             "EDGE_ORPHAN", "STALE", "ATLAS_EMPTY", "SPEC_KIND")
                              for it in ov_issues if it["file"].startswith("03-Atlas/"))
    # ID_NAMESPACE 联邦边界：声明副库中 ov-* 节点豁免（不报 ERROR/WARN）
    ov_ns_files = {
        "00-ROOT.md": ("---\nid: \"spec-root-000\"\nlayer: 3\ntype: spec\nspec-kind: read\n"
                       "project: demo\ndomain: [test]\nmaturity: growing\ncreated: 2026-11-20\n"
                       "last-touched: 2026-11-20\nsummary: root\nid-namespace: demo\n---\n# ROOT\n"
                       "- [r](03-Atlas/README.md)\n"),
        "03-Atlas/README.md": _ov_fix("readme"),
    }
    ov_ns_issues, _, _ = analyze(ov_ns_files)
    ov_ok = ov_ok and not any(it["rule"] == "ID_NAMESPACE" for it in ov_ns_issues)

    ct_files = {"00-ROOT.md": files["00-ROOT.md"], SCHEMA_REL: schema_good}
    ct_issues, _, _ = analyze(ct_files)
    ct_good = not any(it["rule"] == "CONTRACT_DRIFT" for it in ct_issues)
    ct_bad_files = {"00-ROOT.md": files["00-ROOT.md"],
                    SCHEMA_REL: schema_good.replace("  summary:     string\n", "")}
    ct_bad_issues, _, _ = analyze(ct_bad_files)
    ct_bad = any(it["rule"] == "CONTRACT_DRIFT" and it["level"] == "ERROR"
                 and "common 必填字段表" in it["message"] for it in ct_bad_issues)
    ct_gov_bad_files = {"00-ROOT.md": files["00-ROOT.md"],
                        SCHEMA_REL: schema_good.replace("    last-touched: date\n", "")}
    ct_gov_bad_issues, _, _ = analyze(ct_gov_bad_files)
    ct_gov_bad = any(it["rule"] == "CONTRACT_DRIFT" and it["level"] == "ERROR"
                     and "governance.required" in it["message"] for it in ct_gov_bad_issues)
    ct_ov_bad_files = {"00-ROOT.md": files["00-ROOT.md"],
                       SCHEMA_REL: schema_good.replace("    summary:       string\n", "")}
    ct_ov_bad_issues, _, _ = analyze(ct_ov_bad_files)
    ct_ov_bad = any(it["rule"] == "CONTRACT_DRIFT" and it["level"] == "ERROR"
                    and "overview.required" in it["message"] for it in ct_ov_bad_issues)
    contract_ok = ct_good and ct_bad and ct_gov_bad and ct_ov_bad

    # TRIGGER_FM fixtures (v2.14.0, knife F1): red-before-green — missing
    # trigger / missing scope / illegal enum / empty scope each fire ERROR;
    # a compliant pitfall stays clean. In-memory only, same convention.
    def _pit_fix(slug, trig_line, scope_line):
        return ("---\nid: pit-2026-" + slug + "\nlayer: 3\ntype: pitfall\n"
                "project: memory-lib\ndomain: [test]\nmaturity: seed\n"
                "created: 2026-11-20\nlast-touched: 2026-11-20\nsummary: tf\n"
                + trig_line + scope_line + "---\n# PIT-" + slug + "\n")
    tf_files = {
        "00-ROOT.md": files["00-ROOT.md"],
        "01-Specs/PIT-tf-good.md": _pit_fix("tf-good", "trigger: on-domain\n",
                                            'scope: "knowledge-engineering"\n'),
        "01-Specs/PIT-tf-notrig.md": _pit_fix("tf-notrig", "",
                                              'scope: "knowledge-engineering"\n'),
        "01-Specs/PIT-tf-noscope.md": _pit_fix("tf-noscope", "trigger: on-domain\n", ""),
        "01-Specs/PIT-tf-badenum.md": _pit_fix("tf-badenum", "trigger: sometimes\n",
                                               'scope: "knowledge-engineering"\n'),
        "01-Specs/PIT-tf-emptyscope.md": _pit_fix("tf-emptyscope", "trigger: on-domain\n",
                                                  'scope: ""\n'),
    }
    tf_issues, _, _ = analyze(tf_files)
    tf_hits = [it for it in tf_issues if it["rule"] == "TRIGGER_FM"]
    tf_ok = (not any(it["file"] == "01-Specs/PIT-tf-good.md" for it in tf_hits)
             and any(it["file"] == "01-Specs/PIT-tf-notrig.md" and it["level"] == "ERROR"
                     and "缺 trigger" in it["message"] for it in tf_hits)
             and any(it["file"] == "01-Specs/PIT-tf-noscope.md" and it["level"] == "ERROR"
                     and "缺 scope" in it["message"] for it in tf_hits)
             and any(it["file"] == "01-Specs/PIT-tf-badenum.md" and it["level"] == "ERROR"
                     and "非法枚举" in it["message"] for it in tf_hits)
             and any(it["file"] == "01-Specs/PIT-tf-emptyscope.md" and it["level"] == "ERROR"
                     for it in tf_hits)
             and not any(it["rule"] in ("FIELD_MISSING", "TYPE_LAYER", "SPEC_KIND")
                         for it in tf_issues if it["file"].startswith("01-Specs/PIT-tf-")))

    # ATLAS_GROUP fixtures (v2.15.0, knife F2): red-before-green — group missing
    # a card / multi-membership under-listing / unclustered polluted with a hub
    # card / unclustered missing a card / C-1 under-listing / C-1 over-listing
    # each fire WARN; the clean projection stays silent. Domain comes from the
    # SH-test filename + C-1 comment (no hardcoded knowledge-engineering).
    def _ag_card(slug, edges=(), hub=False, domain="[test]"):
        e = ""
        if edges:
            e = "e:\n" + "".join(
                '  - { target: "[[%s]]", rel: %s, weight: 0.7 }\n' % (t, r)
                for t, r in edges)
        return ("---\nid: c-ag-" + slug + "\nlayer: 2\ntype: concept\n"
                "project: memory-lib\ndomain: " + domain + "\nmaturity: seed\n"
                "created: 2026-11-20\nlast-touched: 2026-11-20\nsummary: ag\n"
                + ("hub: true\n" if hub else "") + e + "---\n# " + slug + "\n")

    def _ag_sh(block_lines):
        comment = ("（派生规则：①收录判据 C-1=domain 含 test 的 02-Cards 卡（递归含子目录）；\n"
                   "②分组=hub.abstracts 边，多归属卡每组都列；hub:true 卡自身作组头不重复列入未聚簇。）\n")
        return ("---\nid: s-ag-test\nlayer: 2\ntype: summary\nproject: memory-lib\n"
                "domain: [test]\nmaturity: seed\ncreated: 2026-11-20\n"
                "last-touched: 2026-11-20\nsummary: ag\nedge-policy: exempt\n"
                "---\n# SH-test\n<!-- atlas:auto-start -->\n" + comment
                + "\n".join(block_lines) + "\n<!-- atlas:auto-end -->\n")

    ag_block = [
        "- [[ag-hub-a]]（枢纽）",
        "  - [[ag-inst-a]]",
        "  - [[ag-inst-b]]",
        "- [[ag-hub-b]]（枢纽）",
        "  - [[ag-inst-b]]",
        "  - [[ag-inst-c]]",
        "- 未聚簇：",
        "  - [[ag-free]]",
        "  - [[ag-inc]]",
    ]
    ag_files = {
        "00-ROOT.md": ("---\nid: \"spec-root-000\"\nlayer: 3\ntype: spec\nspec-kind: read\n"
                       "project: memory-lib\ndomain: [test]\nmaturity: growing\n"
                       "created: 2026-11-20\nlast-touched: 2026-11-20\nsummary: root\n"
                       "---\n# ROOT\n- [atlas](03-Atlas/)\n"),
        "03-Atlas/SH-test.md": _ag_sh(ag_block),
        "02-Cards/ag-hub-a.md": _ag_card(
            "ag-hub-a", [("ag-inst-a", "abstracts"), ("ag-inst-b", "abstracts")], hub=True),
        "02-Cards/ag-hub-b.md": _ag_card(
            "ag-hub-b", [("ag-inst-b", "abstracts"), ("ag-inst-c", "abstracts")], hub=True),
        "02-Cards/ag-inst-a.md": _ag_card("ag-inst-a", [("ag-hub-a", "implements")]),
        "02-Cards/ag-inst-b.md": _ag_card(
            "ag-inst-b", [("ag-hub-a", "implements"), ("ag-hub-b", "implements")]),
        "02-Cards/ag-inst-c.md": _ag_card("ag-inst-c", [("ag-hub-b", "implements")]),
        "02-Cards/ag-free.md": _ag_card("ag-free", [("ag-inst-a", "co_occurs")]),
        "02-Cards/incubator/ag-inc.md": _ag_card("ag-inc", [("ag-free", "co_occurs")]),
        "02-Cards/ag-outsider.md": _ag_card(
            "ag-outsider", [("ag-free", "co_occurs")], domain="[other]"),
    }
    ag_clean_issues, _, _ = analyze(ag_files)
    ag_ok = not any(it["rule"] == "ATLAS_GROUP" for it in ag_clean_issues)
    ag_variants = {}
    b = list(ag_block); b.remove("  - [[ag-inst-b]]")   # hub-a group drops multi-member card
    ag_variants["group-missing"] = (dict(ag_files, **{"03-Atlas/SH-test.md": _ag_sh(b)}),
                                    "组缺卡", "ag-inst-b")
    b = list(ag_block) + ["  - [[ag-inst-a]]"]
    ag_variants["unc-polluted"] = (dict(ag_files, **{"03-Atlas/SH-test.md": _ag_sh(b)}),
                                   "混入枢纽卡", "ag-inst-a")
    b = list(ag_block); b.remove("  - [[ag-free]]")
    ag_variants["unc-missing"] = (dict(ag_files, **{"03-Atlas/SH-test.md": _ag_sh(b)}),
                                  "未聚簇组缺卡", "ag-free")
    ag_late = dict(ag_files)
    ag_late["02-Cards/ag-late.md"] = _ag_card("ag-late", [("ag-free", "co_occurs")])
    ag_variants["c1-missing"] = (ag_late, "C-1 漏收", "ag-late")
    b = list(ag_block) + ["  - [[ag-outsider]]"]
    ag_variants["c1-extra"] = (dict(ag_files, **{"03-Atlas/SH-test.md": _ag_sh(b)}),
                               "C-1 多收", "ag-outsider")
    for _name, (_files, _frag, _card) in sorted(ag_variants.items()):
        _issues, _, _ = analyze(_files)
        _hits = [it for it in _issues if it["rule"] == "ATLAS_GROUP"]
        ag_ok = ag_ok and any(it["level"] == "WARN" and _frag in it["message"]
                              and _card in it["message"] for it in _hits)
    # filename fallback: C-1 declared but the domain pattern absent from the
    # comment -> X derives from the SH-{domain} filename segment (SH-test).
    _fb = ag_files["03-Atlas/SH-test.md"].replace(
        "①收录判据 C-1=domain 含 test 的 02-Cards 卡（递归含子目录）；",
        "①收录判据 C-1（域内卡，递归含子目录）；")
    ag_ok = ag_ok and not any(it["rule"] == "ATLAS_GROUP"
                              for it in analyze(dict(ag_files, **{"03-Atlas/SH-test.md": _fb}))[0])
    # out of scope: SH auto block without any C-1 declaration (SH-index form)
    _nc = ag_files["03-Atlas/SH-test.md"].replace(
        "（派生规则：①收录判据 C-1=domain 含 test 的 02-Cards 卡（递归含子目录）；\n"
        "②分组=hub.abstracts 边，多归属卡每组都列；hub:true 卡自身作组头不重复列入未聚簇。）\n", "")
    ag_ok = ag_ok and not any(it["rule"] == "ATLAS_GROUP"
                              for it in analyze(dict(ag_files, **{"03-Atlas/SH-test.md": _nc}))[0])

    issues, counts, broken = analyze(files)
    fired = {it["rule"] for it in issues}
    expect = ["EDGE_ORPHAN", "TYPE_LAYER", "BROKEN_LINK", "FIELD_MISSING",
              "ID_DUP", "LOG_SECTION", "ATLAS_EMPTY", "UNREACHABLE", "STALE", "FM_MISSING",
              "CORR_SECTION", "CORR_TARGET", "SPEC_CHANGELOG",
              "LAYER_BAD", "SPEC_KIND", "SUMMARY_LONG", "DUP_TITLE", "DUP_ALIAS"]
    print("entry_doctor --self-test（内置破坏性 fixture，未写盘）")
    ok = True
    for rule in expect:
        hit = rule in fired
        ok = ok and hit
        print("  [%s] %-14s %s" % ("OK" if hit else "MISS", rule,
                                    "被触发" if hit else "未触发（fixture 或规则有误）"))
    # broken-link targets must land in the broken set (for --use-cli reconcile)
    bl_hit = "NoSuchNoteXYZ" in broken
    ok = ok and bl_hit
    print("  [%s] %-14s %s" % ("OK" if bl_hit else "MISS", "BROKEN_SET",
                                "断链目标已收集" if bl_hit else "broken 集未含 fixture 断链目标"))
    # id-namespace: exactly the wrongly-segmented new card fires; copied + prefixed pass
    ok = ok and ns_ok
    print("  [%s] %-14s %s" % ("OK" if ns_ok else "MISS", "ID_NAMESPACE",
                                "命名空间校验正确（新卡续上游号报错，copied+source 豁免，copied 缺 source 升 WARN）" if ns_ok else
                                "ID_NAMESPACE 触发不符预期"))
    # script-volume: clean volume passes all rules; each VOL_* fires on its broken fixture
    ok = ok and vol_ok
    print("  [%s] %-14s %s" % ("OK" if vol_ok else "MISS", "VOL_RULES",
                                "脚本卷契约正确（干净卷零告警豁免知识字段，VOL_FM/FENCE/SHA256/NAME_DUP/KNOWLEDGE_FIELD 各触发，知识规则不泄漏到卷）" if vol_ok else
                                "VOL_* 触发或豁免不符预期"))
    # VOL_DEPS: good chain exempt; missing/malformed/cyclic deps each fire
    ok = ok and vd_ok
    print("  [%s] %-14s %s" % ("OK" if vd_ok else "MISS", "VOL_DEPS",
                                "依赖图校验正确（合法链豁免，缺失/形态/循环各触发）" if vd_ok else
                                "VOL_DEPS 触发或豁免不符预期"))
    # G20: drifted claims fire ERROR; ambiguous/no-alias WARN; correct + history pass
    ok = ok and g20_ok
    print("  [%s] %-14s %s" % ("OK" if g20_ok else "MISS", "DOC_IMPL_DRIFT",
                                "VERSION 锚点对账正确（漂移ERROR含小写/预发布/路径形态，多版本/无别名WARN，正确链接/历史纯文本豁免）" if g20_ok else
                                "DOC_IMPL_DRIFT 触发不符预期"))
    # v2.7.0 parser layer: phantom fence / EOF fence / empty fields / dup key / subset / md link
    ok = ok and parser_ok
    print("  [%s] %-14s %s" % ("OK" if parser_ok else "MISS", "PARSER_V27",
                                "解析层正确（幻影围栏不吞活链/EOF围栏保持代码/空白null必填判缺/重复键与子集外整块作废/md断链检出）" if parser_ok else
                                "解析层 fixture 不符预期"))
    # CONTRACT_DRIFT: schema↔doctor constants reconciliation (positive + negative)
    ok = ok and contract_ok
    print("  [%s] %-14s %s" % ("OK" if contract_ok else "MISS", "CONTRACT_DRIFT",
                                "契约对账正确（schema 与常量一致豁免，字段表漂移 ERROR）" if contract_ok else
                                "CONTRACT_DRIFT 触发不符预期"))
    # hub: clean hub passes; HUB_GROUNDS/HUB_DECOMPRESSION/SECONDARY_TYPE each fire
    ok = ok and hub_ok
    print("  [%s] BOOL_SCALAR hub:false 解析为假 / \"false\" 保持字符串"
          % ("OK" if bool_ok else "FAIL"))
    ok = ok and bool_ok
    print("  [%s] %-14s %s" % ("OK" if hub_ok else "MISS", "HUB_RULES",
                                "枢纽契约正确（干净hub零告警，入度不足/无abstracts/缺逆边各触发WARN，secondary-type非法/相同ERROR/缺计划WARN）" if hub_ok else
                                "HUB_*/SECONDARY_TYPE 触发不符预期"))
    # governance: clean nodes pass all rules; each GOV_* fires on its broken fixture;
    # knowledge rules do not leak; ID_NAMESPACE exempts gov-* (federation boundary)
    ok = ok and gov_ok
    # STATUS_RULES: size WARN + singleton presence (vault_root=None skips CLAIM)
    status_files = dict(gov_files)  # reuse gov fixtures (has 2 status nodes)
    # Add a fat status body to trigger SIZE
    fat_body = "\n".join("line %d" % i for i in range(210))
    status_files["05-Governance/gov-status-a.md"] = (
        "---\nid: gov-status-a\ntype: status\nstatus: ACTIVE\nscope: workspace-agent\n"
        "created: 2026-11-16\nlast-touched: 2026-11-16\n---\n" + fat_body + "\n")
    st_issues, _, _ = analyze(status_files)
    st_size = [it for it in st_issues if it["rule"] == "STATUS_SIZE"]
    status_ok = len(st_size) >= 1 and st_size[0]["level"] == "WARN"
    # singleton presence: remove all status nodes, set vault_root -> ERROR
    no_st = {k: v for k, v in gov_files.items()
             if "gov-status" not in k}
    ns_issues, _, _ = analyze(no_st, vault_root=".")
    ns_pres = [it for it in ns_issues if it["rule"] == "GOV_SINGLETON"
               and "缺失" in it["message"]]
    status_ok = status_ok and len(ns_pres) == 1
    print("  [%s] %-14s %s" % ("OK" if status_ok else "MISS", "STATUS_RULES",
          "STATUS_SIZE WARN + GOV_SINGLETON presence 正确"))
    ok = ok and status_ok
    print("  [%s] %-14s %s" % ("OK" if gov_ok else "MISS", "GOV_RULES",
                                "治理族契约正确（干净节点零告警豁免知识字段，GOV_FM/STATUS/ENTRY/KNOWLEDGE_FIELD/SINGLETON 各触发，知识规则不泄漏，副库 gov-* 豁免 ID_NAMESPACE）" if gov_ok else
                                "GOV_* 触发或豁免不符预期"))
    # overview: clean README passes all rules (UNREACHABLE exempt); OV_* fire on their
    # broken fixtures; BROKEN_LINK stays ERROR; knowledge rules do not leak; ov-* exempt
    ok = ok and ov_ok
    print("  [%s] %-14s %s" % ("OK" if ov_ok else "MISS", "OV_RULES",
                                "overview 契约正确（干净README零告警+UNREACHABLE豁免，OV_FM/KNOWLEDGE_FIELD 各触发，断链仍ERROR，知识规则不泄漏，副库 ov-* 豁免 ID_NAMESPACE）" if ov_ok else
                                "OV_* 触发或豁免不符预期"))
    # pitfall trigger contract: missing/illegal trigger + missing/empty scope fire;
    # compliant pitfall clean (red-before-green, knife F1)
    ok = ok and tf_ok
    print("  [%s] %-14s %s" % ("OK" if tf_ok else "MISS", "TRIGGER_FM",
                                "pitfall 触发契约正确（缺trigger/缺scope/空scope/非法枚举各ERROR，合规卡零告警）" if tf_ok else
                                "TRIGGER_FM 触发不符预期"))
    # atlas projection reconciliation: five broken projections each fire WARN;
    # clean projection silent (red-before-green, knife F2)
    ok = ok and ag_ok
    print("  [%s] %-14s %s" % ("OK" if ag_ok else "MISS", "ATLAS_GROUP",
                                "SH 投影对账正确（组缺卡/未聚簇混入枢纽卡/未聚簇缺卡/C-1漏收/C-1多收各WARN，干净投影零告警）" if ag_ok else
                                "ATLAS_GROUP 触发不符预期"))
    # reconcile_cli pure-function self-test (no App needed): classification + divergence detection
    notes, warns = reconcile_cli("Foo\n03-Atlas/\n90-Meta/MAP.yaml\n", {"Bar"})
    rc_ok = (len(warns) == 2
             and notes["cli_only_wikilink"] == ["Foo"]
             and notes["offline_only"] == ["Bar"]
             and notes["cli_only_out_of_scope"] == ["03-Atlas/", "90-Meta/MAP.yaml"])
    notes2, warns2 = reconcile_cli("Baz\n", {"Baz"})
    rc_ok = rc_ok and not warns2 and notes2["agree_broken"] == ["Baz"]
    # the empty-result sentinel string must be filtered out, not mistaken for a broken target and raised as a false WARN (pitfall B)
    notes3, warns3 = reconcile_cli("No unresolved links found.\n", set())
    rc_ok = rc_ok and not warns3 and not notes3["cli_only_wikilink"] \
        and not notes3["cli_unresolved"]
    ok = ok and rc_ok
    print("  [%s] %-14s %s" % ("OK" if rc_ok else "MISS", "CLI_RECONCILE",
                                "对账分类/分歧检出正确" if rc_ok else "reconcile_cli 分类或检出不符预期"))
    # the CLI must be pinned to the scanned vault: an error/wrong-vault reply is not an
    # empty link list, and must degrade rather than manufacture a CLI_RECONCILE divergence
    pin_ok = (_cli_file_found("path\t00-ROOT.md\nname\t00-ROOT")
              and not _cli_file_found('Error: File "00-ROOT.md" not found.')
              and not _cli_file_found("")
              and _cli_probe_files({"a/b/c.md": 1, "00-ROOT.md": 1, "z.md": 1})
              == ["00-ROOT.md", "z.md", "a/b/c.md"]
              and vault_display_name(os.path.join("x", "y", "vault")) == "vault"
              and vault_display_name("C:\\a\\b\\vault\\") == "vault"
              and vault_display_name("") == ""
              and _cli_output_unusable("Vault not found.")
              and _cli_output_unusable('Error: Command "unresolved" not found. It may require a plugin to be enabled.')
              and not _cli_output_unusable("No unresolved links found.")
              and not _cli_output_unusable("Foo\nBar\n"))
    ok = ok and pin_ok
    print("  [%s] %-14s %s" % ("OK" if pin_ok else "MISS", "CLI_VAULT_PIN",
                                "vault 定向与错误回复判别正确" if pin_ok else
                                "vault 定向或错误回复判别不符预期"))
    # D10 regression: a CLI reply with Chinese (UTF-8) must decode, not crash via
    # the Windows locale (GBK). Spawns a child emitting UTF-8 Chinese; on a cp936
    # locale this fails if _cli_run loses encoding="utf-8".
    try:
        d10 = _cli_run([sys.executable, "-c",
                        "import sys; sys.stdout.buffer.write('未找到文件'.encode('utf-8'))"], 15)
        d10_ok = (d10.stdout or "").strip() == "未找到文件"
    except Exception:
        d10_ok = False
    ok = ok and d10_ok
    print("  [%s] %-14s %s" % ("OK" if d10_ok else "MISS", "CLI_UTF8_DECODE",
                                "CLI 中文 UTF-8 输出正确解码" if d10_ok else
                                "CLI 中文输出解码失败（D10 回归）"))
    print("\nself-test: %s（触发 %d/%d 规则 + BROKEN_SET/VOL_RULES/GOV_RULES/OV_RULES/TRIGGER_FM/ATLAS_GROUP/HUB_RULES/BOOL_SCALAR/STATUS_RULES/DOC_IMPL_DRIFT/CLI_RECONCILE/CLI_VAULT_PIN/CLI_UTF8_DECODE；fixture 计 %d ERROR / %d WARN / %d INFO）"
          % ("PASS" if ok else "FAIL", len(fired & set(expect)), len(expect),
             counts["error"], counts["warn"], counts["info"]))
    return 0 if ok else 1


# ---------------------------------------------------------------- output

def output(issues, counts, nfiles, root, as_json, mode, cli_reconcile=None):
    if as_json:
        print(json.dumps({
            "tool": "entry_doctor", "version": VERSION, "vault": root,
            "scanned": nfiles, "mode": mode, "counts": counts,
            "issues": sorted(issues, key=lambda x: (x["file"], x["level"], x["rule"])),
            "cli_reconcile": cli_reconcile,
        }, ensure_ascii=False, indent=2))
        return
    print("entry_doctor v%s: 扫描 %d 个 .md（跳过 Templates/ 等）" % (VERSION, nfiles))
    print("模式: %s" % mode)
    if cli_reconcile is not None:
        print("[obsidian unresolved 自动对账]")
        print("  注: %s" % cli_reconcile.get("index_freshness_note", ""))
        print("  一致断链(双方都判断链): %s" % (", ".join(cli_reconcile["agree_broken"]) or "无"))
        print("  离线有 CLI 无(疑离线误报/索引滞后): %s" % (", ".join(cli_reconcile["offline_only"]) or "无"))
        print("  CLI 有 离线无(wikilink, 疑离线漏判): %s" % (", ".join(cli_reconcile["cli_only_wikilink"]) or "无"))
        print("  非 wikilink 目标(目录/markdown 链接, 离线不覆盖): %s"
              % (", ".join(cli_reconcile["cli_only_out_of_scope"]) or "无"))
        print("-" * 40)
    if not issues:
        print("PASS: 0 ERROR / 0 WARN / 0 INFO")
        return
    cur = None
    for it in sorted(issues, key=lambda x: (x["file"], {"ERROR": 0, "WARN": 1, "INFO": 2}[x["level"]], x["rule"])):
        if it["file"] != cur:
            print("\n%s" % it["file"])
            cur = it["file"]
        print("  [%s] %s: %s" % (it["level"], it["rule"], it["message"]))
        print("        修复方向: %s" % it["fix"])
    print("\n合计: %d ERROR / %d WARN / %d INFO（%d 文件）"
          % (counts["error"], counts["warn"], counts["info"], nfiles))


def vault_root(cli):
    here = os.path.abspath(os.path.join(os.path.dirname(__file__), "..", ".."))
    location = resolve_vault(cli, script_fallback=here)
    return location.root, location.source


def main():
    ap = argparse.ArgumentParser()
    ap.add_argument("--vault")
    ap.add_argument("--json", action="store_true")
    ap.add_argument("--use-cli", action="store_true")
    ap.add_argument("--self-test", action="store_true")
    a = ap.parse_args()

    if a.self_test:
        return self_test()

    try:
        root, root_src = vault_root(a.vault)
    except VaultResolutionError as exc:
        print("vault 根解析失败: %s" % exc)
        return 2
    if not os.path.isdir(root):
        print("vault 根不存在: %s（来源: %s；用 --vault 或 $TTL_VAULT 指定 vault 根）"
              % (root, root_src))
        return 2

    files, extra = load_vault(root)
    issues, counts, broken = analyze(files, extra, vault_root=root)

    mode = "离线断链（一致性优先）"
    recon = None
    if a.use_cli:
        cli_out, note = use_cli_crosscheck(root, files)
        if note:
            mode = "离线断链 + CLI 降级（%s）" % note
        else:
            mode = "离线断链 + obsidian unresolved 自动对账"
            recon, warns = reconcile_cli(cli_out, broken)
            for lvl, rule, msg, fix in warns:
                issues.append({"file": "(CLI对账)", "level": lvl, "rule": rule,
                               "message": msg, "fix": fix})
            if warns:
                counts = {"error": 0, "warn": 0, "info": 0}
                for it in issues:
                    counts[it["level"].lower()] += 1

    output(issues, counts, len(files), root, a.json, mode, recon)
    return 1 if counts["error"] else 0


if __name__ == "__main__":
    try:
        sys.exit(main())
    except Exception as e:
        print("entry_doctor 自身失败: %r" % e)
        sys.exit(2)
````
