---
id: "pit-2026-00003"
layer: 3
type: pitfall
trigger: on-domain     # always|on-domain|on-file-pattern|on-project
scope: "memory-governance"   # 涉及"记忆/知识调取"时触发
project: memory-lib
domain: [memory-governance, knowledge-engineering]
maturity: seed
provenance: feedback    # 源自用户当场指正
created: 2026-09-27
last-touched: 2026-11-20
summary: '"调取项目知识/记忆"默认指走 Obsidian 通道从 vault 取，而非查 harness 内置记忆(SearchMemory 只有引脚/通用准则)'
e:
  - { target: "[[SPEC-READ]]", rel: grounds, weight: 0.8 }
  - { target: "[[PIT-私有记忆与引脚混淆]]", rel: co_occurs, weight: 0.6 }
  - { target: "[[00-ROOT]]", rel: grounds, weight: 0.4 }
---

# ⚠️ 记忆调取层级混淆：harness 引脚 ≠ vault 知识本体

## Bug 类型

检索侧的层级误判——用户说"调取/测试记忆"时，Agent 反射性查 harness 内置记忆（`SearchMemory`），而非走 Obsidian 通道从 vault 取知识本体。与 [[PIT-私有记忆与引脚混淆]] 的**删除侧**混淆成对：那条是"删记忆时分不清引脚 vs 知识本体"，这条是"调知识时分不清该查哪层"。

## 根因

两层记忆未在规格里显式区分：

- **harness 内置记忆**：按 D18 仅含引脚 + 通用准则，**不是**项目知识本体；
- **vault**：项目知识本体的唯一权威，须经 Obsidian 通道（CLI `search`/`read`/`backlinks`）调取。

规格只写了"访问通道"（怎么读 vault），没写"调取项目知识 = 走 Obsidian 而非 harness"这条映射，Agent 便走了捷径。

## 修复模式

调取前**探活 → 路由 → 降级**：

1. **探活**：`Get-Process Obsidian` + `obsidian version`；
2. **路由**：App 活 → 走官方 CLI 检索；
3. **降级（D33）**：App 未运行 → 先尽自启义务；MCP/REST/CLI 仍全不可用 → **呈报 Owner 批准后才许退离线兜底**（grep/Python），不得自行降级；**不硬失败、不归咎用户**（App 开关是运行时状态，非用户过错），但未获批准不擅自磁盘读知识。

判据一句话：**调"项目知识"→ 走 Obsidian/vault；查"用户偏好/通用准则/引脚"→ 才用 harness `SearchMemory`。**

## 适用边界

- 有效：本 TTL vault 语境下任何"记忆/知识调取"任务；
- 无效：明确指向 harness 记忆的查询（取用户跨会话偏好、operating-mode 引脚等）——那本就该用 `SearchMemory`。

## 来源

- 真实事件（2026-09-27）：用户要求"测试记忆调取体验"，Agent 误用 `SearchMemory`（只召回引脚/通用准则），经用户"你不应该通过 Obsidian 调用吗"当场指正后纠正。**注**：Agent 并未归咎用户未开 App（明确讲过"根本不怪你"）；harness 事后自动生成的一条同主题影子记忆（`07d4aac5`）曾如此误记，以本卡为准。
- 活体复测背书（Obsidian v1.13.7，PROC=4）：`search query=aliases`→命中 7 文件；`backlinks HippoRAG检索机制`→4（含新卡**嵌套 `e:`** 与 log 的 **`touched:` frontmatter 数组**，证实嵌套对象 + frontmatter 数组内 `[[X]]` 均被链接引擎识别）；`unresolved`→仅 2 项已知（`03-Atlas/`、`90-Meta/MAP.yaml`）。
- 关联裁决：D25（两层调取通道）、D18（测试期知识本体只归 vault）、D20/D21（反链收回 / 边嵌套单轨）。
