---
id: "pit-2026-00007"
layer: 3
type: pitfall
trigger: on-domain
scope: "knowledge-engineering"
project: memory-lib
domain: [knowledge-engineering, obsidian-behavior]
maturity: seed
provenance: manual
created: 2026-09-28
last-touched: 2026-09-28
summary: Obsidian 目录链接不给内部笔记产生反链；doctor 却前缀展开当可达，二者分歧致可达性宽松误判（坑A）
e:
  - { target: "[[entry_doctor校验清单与自检法]]", rel: grounds, weight: 0.7 }
  - { target: "[[PIT-嵌套wikilink识别]]", rel: co_occurs, weight: 0.5 }
---

# ⚠️ 目录链接不产生笔记反链、可达性与 doctor 分歧（坑A）

## Bug 类型

模型分歧——doctor 的可达性判定与 Obsidian 反链引擎对"目录链接"的处理不一致，导致"某笔记可达"是 doctor 单方面宽松判定、Obsidian 里实际不可达（无反链）。

## 根因

- **doctor `compute_reachable`** 对目录链接（如 `03-Atlas/`）做**前缀展开**：把该目录下所有笔记都当作从链接方可达；
- **Obsidian 链接引擎**不给目录链接产生指向其内部笔记的**反链**——本机实测 `obsidian backlinks` 查 `SH-knowledge-engineering` 只有 GraphRAG 卡 + log-03/05，**无 ROOT**。

故"ROOT 经目录链接 `03-Atlas/` 可达 SH"在 doctor 成立、在 Obsidian 反链图里不成立。

## 修复模式

1. 分入口/承重链接用**显式 wikilink**（如 ROOT 写 `[[SH-knowledge-engineering]]`），不依赖目录链接的宽松可达——批次1 已按此修复恢复可达链；
2. doctor 目录前缀展开是有意为之的导航语义（宽松），但须**标注与 Obsidian 反链引擎的分歧**；
3. 待定：doctor 是否在"仅靠目录展开才可达"时告警（留 P1/后续）。

## 适用边界

- 有效：vault 用目录链接（`[[dir/]]` 或 markdown 目录路径）做导航时；本机 Obsidian 1.13.7 实测；
- 深层未闭环：当前库靠 ROOT 显式 wikilink 承重、暂稳；doctor 目录展开与 Obsidian 反链的模型分歧本身仍在（log-2026-09-28-02 记"坑A 仍未闭环"）。

## 来源

- log-2026-09-28-01（坑A 诊断：compute_reachable 前缀展开 vs `obsidian backlinks` 实测无 ROOT）；
- log-2026-09-28-02（批次2 未动、坑A 仍待闭环）。
