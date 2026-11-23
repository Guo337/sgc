---
id: "pit-2026-00008"
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
summary: 外部 CLI"无结果"常输出字面量提示串（如 No unresolved links found.）而非空，对账时须显式过滤，否则误当数据项
e:
  - { target: "[[离线与CLI自动对账模式]]", rel: grounds, weight: 0.7 }
  - { target: "[[entry_doctor校验清单与自检法]]", rel: co_occurs, weight: 0.5 }
---

# ⚠️ 工具/CLI 空结果哨兵串误当数据项（坑B）

## Bug 类型

解析误判——把外部权威 CLI 的"无结果"提示串当成一条真数据，导致集合对账报出假阳性（假 WARN）。

## 根因

`obsidian unresolved` 在**无断链时不是输出空**，而是输出字面量哨兵串 `No unresolved links found.`。`reconcile_cli` 把它当**一个未解析 wikilink 目标**，落入"CLI 有、离线无"分类→升假 `CLI_RECONCILE` WARN。上轮 P0-a 因库里有 2 个真 unresolved 未走到空结果分支，批次1 清净后首次暴露此坑。

## 修复模式

1. 用**独立正则常量** `CLI_EMPTY_RE = ^no unresolved links? found\.?$`（不区分大小写）在收集 CLI 原始输出时过滤哨兵串；
2. 用独立常量而非硬编码字符串——便于未来 CLI 文案变化时**单点维护**；
3. `--self-test` 加空哨兵用例守护（断言无 warns、cli_only_wikilink/cli_unresolved 皆空）。

## 适用边界

- 有效：任何"自研轻量校验 vs 权威工具输出"的交叉核对场景（通用 pitfall）——"无结果"往往不是空输出而是字面量提示串；
- 风险：`CLI_EMPTY_RE` 耦合 obsidian 1.13.7 英文文案；若未来 CLI 改文案或本地化，过滤失效→假 WARN 复现；self-test 只守护正则逻辑、不感知真实 CLI 文案漂移。

## 来源

- log-2026-09-28-01（坑B 暴露：reconcile_cli 把哨兵串当未解析目标）；
- log-2026-09-28-02（doctor v2.2.0 修复：CLI_EMPTY_RE 过滤 + self-test 空哨兵用例；CLI 实测复现前后对比确认）。
