---
id: "pit-2026-00006"
layer: 3
type: pitfall
trigger: on-domain
scope: "knowledge-engineering"
project: memory-lib
domain: [knowledge-engineering]
maturity: seed
provenance: manual
created: 2026-09-28
last-touched: 2026-09-28
summary: 静态检查器把"跳过校验的目录"连带移出"合法链接目标集"，会误判指向模板等的链接为断链
e:
  - { target: "[[entry_doctor校验清单与自检法]]", rel: grounds, weight: 0.7 }
  - { target: "[[PIT-断链检查须剥离代码跨度]]", rel: co_occurs, weight: 0.5 }
---

# ⚠️ 校验集与可解析目标集须分离

## Bug 类型

工具误报——跳过校验的目录（`Templates/`）里的文件被同时从"合法链接目标集"剔除，导致指向它们的 `[[T-log]]`/`[[T-report]]` 被误判断链（BROKEN_LINK）。

## 根因

把两件本应独立的事耦合成一个集合：① **是否校验该文件**（schema/结构合规）② **该文件名是否可作链接目标**（可解析）。跳过 `Templates/` 校验时，连带把模板名移出可解析目标集；而 Obsidian **索引模板为笔记**（Templates 下文件是合法链接目标），于是引用模板的链接被误判断链。

## 修复模式

分离两集合：
- `VALIDATE_SKIP`（如 Templates）：**不校验**，但其文件名仍加入 `extra_resolvable`（**仍是合法链接目标**）；
- `HARD_SKIP`（`.git`/`.obsidian`/`community-research`/`__pycache__` 等）：**既不校验、也不作链接目标**（VCS/配置/第三方快照）。

见 entry_doctor `load_vault` 返回 `(files, extra_names)` 的双集设计。

## 适用边界

- 有效：任何"部分文件跳过校验、但仍可被其他文件引用"的静态检查器；
- 判据一句话：**"跳过校验" ≠ "不存在"**——被跳过的文件是否仍是合法引用目标，须单独判定。

## 来源

- log-2026-09-27-05：entry_doctor v2 首跑即逮到自身此回归（`[[T-log]]`/`[[T-report]]` 误判断链），修为校验集与可解析目标集分离。
