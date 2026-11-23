---
id: "c-2026-00031"
layer: 2
type: concept
project: memory-lib
domain: [memory-governance, knowledge-engineering]
maturity: seed
provenance: manual
created: 2026-11-20
last-touched: 2026-11-20
summary: Agent 身份大类四分（维护者/消费者/联邦/审核者），身份决定入口与义务包；seed 待真实使用验证
aliases: ["Agent身份分类学", "身份分类学", "Agent 身份大类"]
e:
  - { target: "[[SH-knowledge-engineering]]", rel: part_of, weight: 0.7 }
  - { target: "[[SPEC-READ]]", rel: derives_from, weight: 0.6 }
---

# Agent 身份分类学

## 定义

按「来本库做什么」分大类，身份定入口与义务包。同 incubator 哲学，管 Agent 身份而非卡身份。

## 核心

首批四分（seed，Owner 裁）：

- 维护者：改本库→[[role-ke-maintainer]]
- 消费者：只读用知识→[[role-ke-consumer]]
- 联邦：建副库/上提→[[role-ke-federation]]
- 审核者：独立验收/评审（产出≠裁判）→[[role-ke-auditor]]

A/B 二分系 AGENTS.md 实证；consumer/auditor 推设待验。

## 存疑

三分 vs 四分张力已由 Owner 裁四分（2026-11-20）。留存疑：①审核者与维护者临时只读态的边界；②审核者↔SPEC-GOVERN「审核=并入 Agent」合一态的映射与 merge 权归属未裁（宿主握 merge 权，role-ke-auditor 仅评审面；上提验收流程卡验收者=复核+并入是第三口径）——Owner 裁 C 案等真实触发（2026-11-20：审核者系备用角色，短期不进一步开发）；类集仍开放，演化归 Owner。

## 应用

G2-3 据此建四角色件；SH-index 据此写身份跳转祈使句。

## changelog

- 2026-11-20：建档（G2-1）。
- 2026-11-20：Owner 裁四分，增审核者类（G2-1 v2）。
- 2026-11-20：存疑节补②合一态映射/merge 权张力（log-04 宣称入册实缺，独立分析抓出；Owner 裁 C 案等真实触发，审核者系备用角色短期不开发）；role-ke-auditor 分权指针加现状限定半句（corr-2026-11-20-83）。