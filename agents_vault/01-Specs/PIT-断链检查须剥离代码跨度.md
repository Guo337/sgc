---
id: "pit-2026-00005"
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
summary: 自研离线断链正则若扫全文提取双方括号链接，会把代码跨度/围栏块里的示例链接误报为断链
e:
  - { target: "[[entry_doctor校验清单与自检法]]", rel: grounds, weight: 0.7 }
  - { target: "[[PIT-嵌套wikilink识别]]", rel: co_occurs, weight: 0.4 }
---

# ⚠️ 断链检查须剥离代码跨度/围栏块

## Bug 类型

工具误报——离线断链扫描把文档里**代码内的示例链接**（写在行内代码或围栏块中、形如双方括号包裹的目标名）当成真链接，报出假断链（BROKEN_LINK）。

## 根因

自研正则扫全文提取双方括号链接，未排除行内代码（反引号包裹）与围栏代码块（三反引号 / 三波浪号包裹）。而 Obsidian 语义是**“代码内不成链接”**（D21）——写在代码跨度里的链接示例只是字面文本、不产生链接。正则未对齐此语义，遂把示例误当断链目标。

## 修复模式

提取 wikilink 前**先剥离代码**：用 `FENCE_RE`（围栏块）+ `INLINE_CODE_RE`（行内代码）把代码跨度替换为空白，再在剩余文本上跑 wikilink 正则（见 entry_doctor `strip_code`）。这样对齐 Obsidian"代码内不成链接"语义，消除示例误报。

## 适用边界

- 有效：任何**自研离线**链接/断链扫描器（不依赖 Obsidian 引擎）；
- 无效：走 Obsidian 原生 CLI（`obsidian unresolved`）的路径——原生已正确处理代码内链接，无需自己剥离；
- 关联教训：离线正则只做 wikilink 解析、非重造完整链接引擎（决策 B 边界）。

## 来源

- log-2026-09-27-04：entry_doctor 首跑即逮到自身此误报，修为提取前剥离围栏块与行内代码（对齐 Obsidian 语义，D21 命门正向利用）。
- 自指实证（本轮）：本卡初版正文写了字面双方括号示例 + 围栏样例，围栏与行内反引号配对被打破、示例未被剥离，反被 doctor 判为 2 条真断链——遂改为文字描述、不留裸双方括号，正是本坑的活体复现。
