---
id: "c-2026-00009"
layer: 2
type: procedure
project: memory-lib
domain: [knowledge-engineering]
maturity: seed
provenance: reflect
created: 2026-09-28
last-touched: 2026-11-16
summary: 自研轻量校验与权威工具对账：按可比性分类避免假阳性、真分歧升非阻断 WARN、空结果哨兵须过滤
k: []
v: []
e:
  - { target: "[[SH-knowledge-engineering]]", rel: part_of, weight: 0.7 }
  - { target: "[[记忆通道使用纪律]]", rel: implements, weight: 0.6 }
  - { target: "[[entry_doctor校验清单与自检法]]", rel: co_occurs, weight: 0.6 }
  - { target: "[[entry_doctor]]", rel: co_occurs, weight: 0.5 }
aliases: ["离线CLI对账", "自动对账模式"]
---

# 离线与CLI自动对账模式

## 定义

当自研轻量校验器（离线扫描）与权威工具（第一方 CLI）就同一问题各给答案时，按"可比性"分类对账、而非裸集合相减，以区分真分歧与语义差。

## 核心

reflect 编译自候选结晶（来源：log-2026-09-27-07；坑B 修正见 log-2026-09-28-02）：

1. **按可比性分类**：离线只扫 `[[wikilink]]`，CLI `unresolved` 还含目录/markdown 链接/embed；故分四类——① 一致断链 ② 离线有 CLI 无(疑离线误报/索引滞后) ③ CLI 有离线无(疑离线漏判) ④ 非 wikilink 目标(离线不覆盖，仅提示)；
2. **真分歧升非阻断 WARN**：②③ 升 `CLI_RECONCILE`(WARN，不阻断 merge)；④ 不入闸门——避免把语义差误报成断链分歧；
3. **空结果哨兵须过滤**（坑B）：权威工具"无结果"常是字面量提示串（如 `No unresolved links found.`）而非空输出，须显式过滤，否则误当数据项升假 WARN；
4. **优雅降级**：CLI 不可用（App 未运行/超时）则退离线，报告标注降级原因。

## 存疑

- 可比性判据是启发式（尾斜杠/路径分隔/扩展名）：含路径的 wikilink `[[folder/note]]` 会被误归 ④；当前库 wikilink 皆裸名，无影响。
- 哨兵过滤与 CLI 文案耦合，版本/本地化变化时可能失效。

## 应用

entry_doctor v2.2.0 的 `--use-cli` 即此模式实现；可复用于任何"自研轻量校验 vs 权威工具"的交叉核对场景。
