---
id: "c-2026-00006"
layer: 2
type: procedure
project: memory-lib
domain: [knowledge-engineering, memory-governance]
maturity: seed
provenance: reflect
created: 2026-09-28
last-touched: 2026-11-24
summary: entry_doctor 作 merge 闸门的检查清单(rule+level+fix，G20 VERSION 锚点)、完整性四分与 --self-test 破坏性 fixture 自检法
k: []
v: []
e:
  - { target: "[[SH-knowledge-engineering]]", rel: part_of, weight: 0.7 }
  - { target: "[[SPEC-GOVERN]]", rel: implements, weight: 0.7 }
  - { target: "[[EFG三原理]]", rel: implements, weight: 0.7 }
  - { target: "[[离线与CLI自动对账模式]]", rel: co_occurs, weight: 0.5 }
  - { target: "[[entry_doctor]]", rel: implements, weight: 0.8 }
aliases: ["doctor校验清单", "entry_doctor检查项"]
---

# entry_doctor校验清单与自检法

## 定义

entry_doctor（只读、纯标准库）作为并入 main 的 CI 闸门，其检查清单与"破坏性 fixture 自检"方法的可复用规范。

## 核心

reflect 编译自候选结晶（来源：log-2026-09-27-02、log-2026-09-27-03、log-2026-09-27-04、log-2026-09-27-05）：

1. **检查清单**（[[entry_doctor#VERSION|v2.17.0]]，每项带 `rule` id + `level`(ERROR/WARN/INFO) + `fix`(修复方向)）：frontmatter 可解析（严格扁平子集，重复键/子集外形态整块作废；无引号 true/false 解析为布尔，`hub: false` 不再误判枢纽，corr-2026-11-18-65）、common 必填（空白/null/空列表判缺）、id 唯一+格式、type↔layer 合法、边完整性、Atlas auto 区块非空、断链（wikilink + markdown .md 双形态）、不可达、stale、重复、日志四段、更正三段、规格禁 changelog、id 命名空间、脚本卷 VOL_* 替代判据、卷依赖图 VOL_DEPS、治理族 GOV_* 替代判据、全景概览 OV_* 替代判据、项目族 PROJ_* 替代判据（FM/KNOWLEDGE_FIELD/PLACEMENT/ENTRY/EDGE_BAN 五件）+ REPORT_FM 申报必填（n/a 合法；类别五分 knowledge/volume/governance/overview/project，corr-2026-11-24-110）、枢纽 HUB_* 解压对账（abstracts↔implements 互逆）+ SECONDARY_TYPE 双 type 过渡、pitfall TRIGGER_FM 触发契约（缺 trigger/scope 或非法枚举 ERROR，scope 首版只判非空，v2.14.0 corr-95）、SH 投影 ATLAS_GROUP 对账（C-1 形态三件：枢纽组↔abstracts 边/未聚簇差集含 2A 豁免/C-1 全集，失配 WARN=重跑 atlas_fill，v2.15.0 corr-96）、实现漂移对账 G20（VERSION 锚点）、契约本体对账 CONTRACT_DRIFT；
2. **完整性四分**（勿混用）：orphan 孤岛(无边) / broken-link 断链(目标不存在，`obsidian unresolved` 只报这类) / unreachable(ROOT 不可达) / stale(仅 L2)；
3. **剥离代码跨度**：断链扫描先剥行内代码/围栏块（CommonMark 子集行扫描：开围栏须行首、info string 禁反引号、闭围栏长度≥开围栏、未闭合延伸到 EOF），对齐 Obsidian 链接语义，否则文档示例 wikilink 误报为断链、幻影围栏吞活链；
4. **--self-test 破坏性 fixture 法**：内存构造故意违规的 fixture，断言每条规则都能被触发（不写盘、无需 App），借 SwarmVault smoke / Loreweave 回归思路。

## 存疑

- 检查清单随 schema/规格演进增删，卡内清单是 v2.x 快照，以 SPEC-GOVERN §doctor 闸门 + 脚本卷 [[entry_doctor]]（真值源，.py 无常驻文件、用时抽取）为准。

## 应用

本库 merge 前必过此闸门（SPEC-GOVERN §doctor 闸门）；"破坏性 fixture 自检"可迁移到任何规则型校验器。
