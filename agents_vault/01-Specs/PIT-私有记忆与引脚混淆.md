---
id: "pit-2026-00001"
layer: 3
type: pitfall
trigger: on-project     # always|on-domain|on-file-pattern|on-project
scope: "memory-lib"     # 作用于本库治理：Agent 评估自身记忆条目去留时
project: memory-lib
domain: [knowledge-engineering, memory-governance]
maturity: seed
provenance: feedback    # 源自一次真实误判（差点删掉引脚）
created: 2026-09-26
last-touched: 2026-09-26
summary: 评估删除 Agent 记忆前，先辨内容是指针(引脚,该留)还是知识本体(私有记忆)；混淆会误删引导机制
e:
  - { target: "[[SPEC-WRITE]]", rel: grounds, weight: 0.6 }
  - { target: "[[00-ROOT]]", rel: grounds, weight: 0.5 }
---

# ⚠️ 私有记忆与引脚混淆陷阱

## Bug 类型

Agent 记忆治理概念混淆 —— 把"引脚"当成"私有记忆"，导致差点误删引导指针。

## 根因

将两类内存条目混为一谈：

- **引脚**：pointer + obligation（库路径、启动/记入义务、operating-mode 指针），**不含知识本体**，是 vault 的引导机制；
- **私有记忆**：persistent knowledge body（领域事实/结论/经验），是知识本体的影子副本。

未领会本库设计约束："harness 内置记忆只作引脚，不存知识本体；知识以总库为唯一权威"（设计文档 §3.1、D10）。

## 修复模式

删除任何记忆前，先核验内容性质：

1. 仅含路径/义务/触发条件等指针信息 → 属引导机制（引脚），**保留**；
2. 含领域事实/结论/经验 → 属私有记忆（知识本体），按"知识归 vault"原则迁移入库后再从内存清除。

判据一句话：**问它是"指向知识"还是"就是知识"**。

## 适用边界

- 有效：本 TTL vault 语境下评估 Agent 内存条目去留时；
- 无效：本库之外的一般记忆管理（通用行为准则、跨会话用户偏好不在此列，它们不是项目知识本体）。

## 来源

- 真实事件：本项目建设期一次误判（原存于 Agent 私有记忆 `a1e57356`，按测试约束"知识本体只归 vault"迁移至此卡，Git 兜底）。
- 关联裁决：D10（内置记忆=引脚）、D18（测试期 Agent 不存私有知识本体，Pi Agent 为天然范例）。
