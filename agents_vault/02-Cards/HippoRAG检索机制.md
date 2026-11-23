---
id: "c-2026-00001"
layer: 2
type: concept
project: memory-lib
domain: [knowledge-engineering, retrieval]
maturity: growing
provenance: manual
created: 2026-09-25
last-touched: 2026-11-20
summary: 以知识图谱为海马体索引、Personalized PageRank 做模式补全，单步完成多跳检索
k: []
v: []
e:
  - { target: "[[GraphRAG层次社区摘要]]", rel: contrasts_with, weight: 0.7 }
  - { target: "[[SPEC-READ]]", rel: grounds, weight: 0.9 }
aliases: ["HippoRAG", "海马体检索", "PPR检索"]
---

# HippoRAG检索机制

## 定义

神经生物学启发的 LLM 长期记忆检索架构（NeurIPS 2024，Ohio State + Stanford）：知识图谱充当海马体索引，Personalized PageRank（PPR）充当模式补全。

## 核心

1. **离线索引**：LLM 抽取三元组建 KG；检索编码器为相似实体加同义边；
2. **在线检索**：查询 → 提取实体 → 匹配图节点（查询节点=部分线索）→ PPR 以查询节点为种子扩散概率 → 节点概率 × 节点-段落矩阵 → 排序；
3. **节点特异性** \(s_i = |P_i|^{-1}\)：出现在越少段落的节点权重越高（IDF 的神经生物学替代）；
4. 实证：多跳问答超传统 RAG 20%+；衍生工作 TERAG 证明 PPR 可省 80% token。

## 存疑

- 来源为技术社区详解转述（S级），论文原文未读；PPR 参数（阻尼系数、种子权重分配）需在检索实验中自行标定。

## 应用

本库应用实例见 [[log-2026-11-20-02]]

## changelog

- 2026-11-20：G1-2 原地剥离升迁普适（移日志案，Owner 签）：核心节 2 处本库实现插入语删除、存疑节 Stage 1→检索实验（存疑事实保留）；应用节原文移 [[log-2026-11-20-02]] 留一行正文指针；frontmatter 边零变化（corr-2026-11-20-75）。
