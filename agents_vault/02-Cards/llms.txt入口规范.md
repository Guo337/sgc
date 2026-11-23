---
id: "c-2026-00003"
layer: 2
type: concept
project: memory-lib
domain: [knowledge-engineering, entry-design]
maturity: growing
provenance: manual
created: 2026-09-25
last-touched: 2026-11-20
summary: 面向LLM的指针式根入口社区标准：H1+摘要+H2链接分区+Optional，是指针不是展开工具
k: []
v: []
e:
  - { target: "[[00-ROOT]]", rel: grounds, weight: 0.95 }
  - { target: "[[GraphRAG层次社区摘要]]", rel: co_occurs, weight: 0.5 }
aliases: ["llms.txt", "llmstxt", "AI入口文件"]
---

# llms.txt入口规范

## 定义

Jeremy Howard 2024-09 提出的社区提案（llmstxt.org，非 W3C 正式标准）：在根路径放一个纯 Markdown 文件，告诉 LLM"这是什么、哪些内容最重要、先读哪里"。Anthropic 官方文档为采用典范（其 llms.txt 约 8.4K token）。

## 核心

1. **结构四要素**：唯一 H1（名称）+ blockquote 摘要（全文权重最高文本）+ 可选自由段落 + H2 链接分区（`- [名称](路径): 一句话说明`）；`## Optional` 分区=上下文预算紧张时可跳过；
2. **V2 核心理念**："llms.txt 本身只是指针，不是展开工具"——Agent 打开或搜索它，然后跟随链接取详情；
3. **子路径规则**：子路径的 llms.txt 覆盖其路径，最具体者优先；
4. **社区经验预算**：≤10K token，"是导览不是目录"；配套 `.md` 后缀约定提供页面纯净版。

## 存疑

- 属社区提案而非强制标准，各 Agent 不会自动读取——须由宿主以启动义务保证入口被读，不依赖 Agent 自发行为。

## 应用

本库应用实例见 [[log-2026-11-20-02]]

## changelog

- 2026-11-20：G1-2 原地剥离升迁普适（移日志案，Owner 签主案）：存疑节"本库靠引脚…保证 ROOT 被读"改写为一般机制"须由宿主以启动义务保证入口被读"（社区提案非强制标准+不自动读取两项存疑事实原样保留，存疑不洗白，独立审核裁定通过）；应用节原文移 [[log-2026-11-20-02]] 留一行正文指针；frontmatter 边零变化，grounds→ROOT 骨架边保留（corr-2026-11-20-75）。
