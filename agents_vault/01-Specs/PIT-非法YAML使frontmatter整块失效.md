---
id: "pit-2026-00013"
layer: 3
type: pitfall
trigger: on-domain     # always|on-domain|on-file-pattern|on-project
scope: "knowledge-engineering"   # 写卡片 frontmatter / 排查链接与反链缺失时触发
project: memory-lib
domain: [knowledge-engineering, obsidian-behavior]
maturity: seed
provenance: manual     # 源自本机活体对账实测
created: 2026-11-07
last-touched: 2026-11-07
summary: 双引号标量闭合后带裸文本 → frontmatter YAML 非法 → Obsidian 丢弃整块属性、e 边与反链全部消失；宽松解析器仍读得出 → 离线对账出假分歧
e:
  - { target: "[[PIT-嵌套wikilink识别]]", rel: grounds, weight: 0.8 }
  - { target: "[[PIT-记忆调取层级混淆]]", rel: co_occurs, weight: 0.4 }
  - { target: "[[SPEC-READ]]", rel: grounds, weight: 0.5 }
---

# ⚠️ 非法 YAML：一个引号让整块 frontmatter 与它的边一起消失

## 症状

- `vault_read` 带 `targetType=frontmatter` 返回
  `Could not parse document frontmatter: Unexpected scalar at node end at line 12, column 21`；
  同一文件全文读取时 `frontmatter` 是空对象。
- 该文件 `e:` 里的 `[[X]]` **全部**从 `links` 消失，指向笔记的 `backlinks` 也不再含它；正文里的 `[[X]]` 照旧计数。
- 离线解析器（宽松正则）仍能读出 `e` 的全部边 → 离线比原生"信息更多" → 对账出现假分歧（曾被误判为索引陈旧/口径不同）。

## 根因

`summary: "调取项目知识/记忆"默认指走…` —— 双引号标量**闭合后仍有裸文本**，YAML 非法。

Obsidian 用真 YAML 解析器，**整块 frontmatter 作废**（不是逐行容错、也不是只丢单个字段）：

- 属性面板 / Bases 无字段；
- 链接引擎虽扫描全文，但该文件的 frontmatter 没被解析成属性，嵌套 `e` 里的 `[[X]]` 随之丢失。

## 二次伤害

`entry_doctor` 的 `FM_MISSING` 语义是"缺失**或不可解析**"，但宽松解析器把非法 YAML 当合法解析掉 → 该规则永不触发 → 全库体检 `0 error / 0 warn` 却藏着非法 frontmatter，缺陷直接穿过闸门。

## 修复模式

1. **写卡**：值内含双引号短语时整值换单引号 —— `summary: '"短语"后续文字'`，文字一个不改。
2. **解析器**：与 Obsidian 同判 —— 引号未闭合、或闭合后仍有裸文本 → 整块 frontmatter 作废（`entry_doctor._scalar_unparseable`），不要"尽量读出一点"。
3. **探测**：`vault_read targetType=frontmatter` 是最便宜的 YAML 合法性探针，比全文读取更早暴露解析错误。
4. **对账**：离线 links 比原生**多**时，先怀疑"原生解析失败"，再怀疑"口径不同"。

## 与既有结论的关系

[[PIT-嵌套wikilink识别]] 的"嵌套 wikilink 被链接引擎识别"成立，但**前提是 YAML 可解析**；本条补上这个前提，不推翻它，只给它加边界条件。

## 适用边界

- 有效：Obsidian 1.13.x 的 frontmatter 解析（本机活体实测）；任何依赖 frontmatter 的脚本解析器。
- 无效：纯正文 markdown 工具（不涉及 frontmatter）。
- 可迁移教训：**宽松解析 ≠ 兼容**。兜底实现若比权威实现"更能读"，就会把权威实现的数据缺陷翻译成"实现分歧"，掩盖真因。

## 来源

- 本机活体实测（2026-11-07）：全库 64 个 md 逐一 `vault_read` 与离线 `OfflineReader` 对账；修复前 links 7 / backlinks 2 / unresolved 5 分歧，仅修 YAML 后 6/0/5，目标集对齐后 0/0/0。
- 关联：[[PIT-嵌套wikilink识别]]、[[PIT-记忆调取层级混淆]]（缺陷文件本身）、[[SPEC-READ]] v0.6.3。
