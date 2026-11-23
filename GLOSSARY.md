# Glossary / 术语对照

> English ↔ Chinese term map for **Stratified Governed Convergence** (internal codename **TTL**).
> The authoritative protocol texts are in Chinese; this glossary lets English readers follow them.
> 权威协议正文为中文，本表供英文读者对照跟随。Terms are kept consistent across the vault.

## Core proposition / 核心命题

| English | 中文 | Meaning |
|---|---|---|
| Stratified Governed Convergence (SGC) | 成层治理收敛 | External project name; three words each carry one philosophical pillar |
| certainty convergence | 确定性收束 | Alpha milestone: convergence is deterministic, honest, reproducible, auditable — not a guarantee that every conclusion is correct |
| semantic convergence | 语义收敛 | Core hypothesis (under verification): AI reads raw L1 memory and converges it into stable L2/L3 knowledge |
| Anti-Convergence | 反收敛 | Discipline of NOT collapsing uncertainty prematurely; "I don't know" is a first-class citizen |
| epistemic honesty | 认知诚实 | Preserve uncertainty, preserve provenance, preserve human adjudication; never give false certainty |

## Layers / 分层

| English | 中文 | Meaning |
|---|---|---|
| L1 (episodic) | L1 情景层 | Append-only logs & corrections; immutable source; corrections are new entries back-referencing, never overwrites |
| L2 (semantic) | L2 语义层 | Mutable knowledge cards crystallized from L1 |
| L3 (specification) | L3 规格层 | Authoritative specs/rules; changes require Owner adjudication |
| stratification | 成层/沉积 | Sediment-like layers: immutable base constrains upper layers; each layer has a different modification contract |
| provenance chain | 来源链 | Truth is a chain, not a snapshot; every conclusion traces back to L1 evidence |

## Governance / 治理

| English | 中文 | Meaning |
|---|---|---|
| governance-as-text | 治理即文本 | Governance lives as natural-language contracts in the same vault/format as knowledge — not as DB/code |
| separation of powers | 分权 | Semantic / skeleton / authority responsibilities separated; code must not guess semantics for AI |
| Owner adjudication | Owner 裁决 | Human ruling cannot be inferred from factual statements; facts do not constitute authorization |
| constraint escape | 约束逸出 | A violation where an agent skips a binding discipline (e.g. skipping the SH walk) |
| no-overreach discipline | 不越权纪律 | Proposal → decision → execution is one-way; a node must not self-promote |
| merge gate | merge 闸门 | doctor runs as the pre-merge gate; branch → review → main |

## Structure & tooling / 结构与工具

| English | 中文 | Meaning |
|---|---|---|
| vault | 记忆库 | Markdown + Obsidian + Git as the single source of truth |
| card | 卡片 | Atomic knowledge unit with frontmatter (k/v/e typed edges) |
| hub card | 枢纽卡 | Navigation hub (hub:true); group head in SH projections |
| SH (sub-entry hub) | 分入口 | Domain/identity navigation entry; progressive disclosure |
| typed edge | 类型边 | e={target, rel, weight}; rel enum (depends_on/derives_from/supersedes/grounds…) |
| nested single-track | 嵌套单轨 | [[X]] embedded in edges reuses Obsidian native backlinks/graph/search; no separate graph engine |
| anti-reification | 反物化 | Representation ≠ the thing itself; retrieval ≠ semantic correspondence, graph edge ≠ semantic relation, storage ≠ memory lifecycle |
| script volume | 脚本卷 | Code whose single source is a Markdown volume; .py is a transient extracted artifact, deleted after use |
| byte fidelity | 字节保真 | Binary extraction guarantees byte-exact round-trip (a property of the extractor, not of Markdown) |
| doctor | 校验器 | entry_doctor: mechanical structural/governance guard (rule count truth = volume RULES registry, not materialized in docs), thin skeleton not semantic engine |
| falsifiability | 可证伪 | Core claims frozen as testable specs; blind state, Gold set, mechanical scoring, failure attribution |
| federation / sub-vault | 联邦/副库 | Autonomous project sub-vaults; copied cards keep original id (provenance:copied) |

## Reading order / 阅读顺序

1. [README.md](README.md) — facade & pointer
2. [agents_vault/03-Atlas/README.md](agents_vault/03-Atlas/README.md) — full project map (Chinese, authoritative philosophy host)
3. [AGENTS.md](AGENTS.md) — agent entry
4. Specs: `agents_vault/01-Specs/SPEC-READ|WRITE|GOVERN.md` (Chinese, runtime authority)