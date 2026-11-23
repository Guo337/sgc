---
id: "pit-2026-00002"
layer: 3
type: pitfall
trigger: on-domain     # always|on-domain|on-file-pattern|on-project
scope: "knowledge-engineering"   # 设计/修改 frontmatter 边表示时触发
project: memory-lib
domain: [knowledge-engineering, obsidian-behavior]
maturity: seed
provenance: manual     # 源自本机活体实测，非微词反馈
created: 2026-09-26
last-touched: 2026-09-26
summary: Obsidian 嵌套 frontmatter 的 wikilink 被链接引擎识别(反链/图谱)，但嵌套字段 rel/weight 不可被 Bases 查询
e:
  - { target: "[[HippoRAG检索机制]]", rel: grounds, weight: 0.7 }
  - { target: "[[SPEC-READ]]", rel: co_occurs, weight: 0.4 }
---

# ⚠️ 嵌套 frontmatter wikilink：链接可见 ≠ 属性可查

## Bug 类型

Obsidian frontmatter 数据建模误判——把"属性类型系统不支持嵌套"错误外推为"链接引擎也看不见嵌套里的 `[[wikilink]]`"，据此差点把边表示拆成不必要的双轨（扁平链接列 + e-meta 语义）。

## 根因

Obsidian 有两套**独立**子系统，对"嵌套对象"的处理不同：

- **链接引擎**（backlinks / graph / `file.links`）：扫描全文（含 frontmatter 任意深度）的 `[[...]]` 文本，**嵌套对象内的 `[[X]]` 照样识别**——生成反链、进图谱、被 search 命中；
- **属性类型系统**（Properties 面板 / Bases / Dataview）：官方 `Properties.md` 明列 "Not supported → Nested properties"，指嵌套字段**不能作为结构化属性**被展示或经 `filter`/`formula` 查询。

误把后者的"不支持"套到前者，得出"嵌套链接不可见"的假命门。

## 修复模式

1. 边表示**保持嵌套单轨** `e: [{target:"[[X]]", rel, weight}]` 即可：`[[X]]` 白嫖原生反链/图谱，rel/weight 供脚本与人读；
2. **仅当** Stage 2 需要在 Bases/Dataview 里按 `rel`/`weight` 过滤边时，才额外补一个扁平副本字段（如 `e-rel: [contrasts_with, ...]`），并用 entry_doctor lint 校验两者一致；
3. 判据一句话：**要"链接/反链/图谱" → 嵌套就够；要"Bases 按字段查" → 才需扁平副本。**

## 适用边界

- 有效：Obsidian（本机 1.13.7 实测）vault 内 frontmatter 建模、边表示设计；
- 无效：其他 markdown 工具（不保证同样解析 frontmatter 内链接）；
- **可迁移教训**：对"命门"级架构前提，**实测优先于文档推断**——文档说的"不支持"可能只指某个子系统，需精确区分是哪个引擎。

## 来源

- 本机活体实测（2026-09-26，Obsidian v1.13.7）：`obsidian backlinks file="HippoRAG检索机制"` 返回 `GraphRAG层次社区摘要`；核对 GraphRAG 卡原文，其对 HippoRAG 的**唯一**引用就在嵌套 `e:` 对象内（正文无扁平链接）→ 证实嵌套 wikilink 被链接引擎识别。
- 更正对象：上一轮基于官方 `Properties.md` "Nested properties not supported" 的错误外推（原拟拆双轨）；见设计文档 D21 修订。
- 关联裁决：D21（边保持嵌套单轨）、D20（反链收回可只读消费）、D24（CLI 官方性 + 实测）。
