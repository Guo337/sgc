---
id: "c-2026-00002"
layer: 2
type: concept
project: memory-lib
domain: [knowledge-engineering, retrieval]
maturity: growing
provenance: manual
created: 2026-09-25
last-touched: 2026-11-20
summary: 图社区层次聚类（Leiden）+逐级LLM摘要，全局问题读摘要省97% token
k: []
v: []
e:
  - { target: "[[HippoRAG检索机制]]", rel: contrasts_with, weight: 0.7 }
  - { target: "[[SH-knowledge-engineering]]", rel: part_of, weight: 0.8 }
aliases: ["GraphRAG", "社区摘要", "Leiden聚类"]
---

# GraphRAG层次社区摘要

## 定义

微软研究院的图增强检索方案（arXiv:2404.16130，2024-04 开源）：实体图上做 Hierarchical Leiden 社区检测，逐级生成 LLM 社区摘要，全局问题走摘要 Map-Reduce 而非遍历原文。

## 核心

1. **索引**：实体/关系抽取 → 图构建（同名实体跨文档归一合并）→ Leiden 层次社区划分（模块度优化，层级=缩放粒度）→ 每社区报告式摘要；
2. **双查询模式**：局部问题=实体查找+1–3 跳边扩展；全局问题=社区摘要分批 Map-Reduce（每批 ~4 社区并行提炼，按规模/包含度加权汇总）；
3. 实证：最高层摘要比直接处理源文本省 97% token；
4. 主题由**结构自动涌现**而非关键词预设——这是它优于手工分类的核心。

## 存疑

- 全量重建的 LLM 调用成本高；可行缓解：只重算受影响社区（增量更新），并降低摘要编译频率。

## 应用

本库应用实例见 [[log-2026-11-20-02]]

## changelog

- 2026-11-20：G1-2 原地剥离升迁普适（移日志案，Owner 签）：存疑节"本库对策"改通用缓解表述（成本存疑保留，"每周"实例参数日志留痕）；应用节原文移 [[log-2026-11-20-02]] 留一行正文指针；frontmatter 边零变化，part_of→SH 骨架边保留（corr-2026-11-20-75）。
