---
id: "c-2026-00035"
layer: 2
type: observation
project: memory-lib
domain: [memory-governance, knowledge-engineering]
maturity: seed
provenance: reflect
created: 2026-11-20
last-touched: 2026-11-20
summary: 起草→独立审核→Owner 签三层分权两波实证有真实拦截价值；裁决点之间须留 Owner 微调时窗
k: []
v: []
e:
  - { target: "[[SH-knowledge-engineering]]", rel: part_of, weight: 0.6 }
  - { target: "[[EFG三原理]]", rel: derives_from, weight: 0.6 }
  - { target: "[[外部评审三层分诊法]]", rel: co_occurs, weight: 0.5 }
aliases: ["三层分权", "起草审核签分权"]
---

# 起草-审核-Owner 签三层分权在剥离场景产生真实拦截

## 定义

产出（起草 Agent）→ 独立复核（审核 Agent，非产出方）→ 裁决（Owner 逐件签）的三层分权流程；审核独立性不是仪式，是呈签前的真实拦截层。

## 核心

reflect 编译自候选结晶（来源日志：log-2026-11-20-03），G1 实证+G2 复验：

- **G1 拦截实证**：四卡剥离草稿经独立审核抓出 4 处须修（无据频率断言/惯例论据失实/不存在的卷论据/名单数值矛盾），全部在呈 Owner 前拦下——若直呈，Owner 须自己当审核者。
- **G2 复验**：起草方抓 doctor 区块前缀坑；编排方独立复核 31+35 项；遗留项双线分析再抓出"入册宣称与卡内事实不符"（编排方自己的疏漏被独立线抓出——审核对象包括编排方自身产出）。
- **裁决点留微调时窗**（Owner 流程反馈，2026-11-20）：规划批准≠开工令；呈批后留 Owner 调整窗口再执行——四分推翻三分推荐即微调窗口兑现的实证。裁决点之间不留时窗=Owner 失去修正结构假设的机会。
- 与 F 原理（产出≠裁判）的关系：F 是原理，本卡是流程落地实证；审核者身份的义务包见 [[role-ke-auditor]]（merge 权归属待角色实例化裁决）。

## 存疑

- 审核 Agent 与起草 Agent 同底座模型——独立性是**上下文隔离**级（不见对方推理过程），非异构裁判级；同源盲区（双方共享的错误假设）拦不住，Owner 签是最后防线。
- 拦截样本集中在文本/论据类缺陷；数值/逻辑类缺陷的拦截率未单独统计。

## 应用

本库治理波的标准流程（起草任务书→独立审核任务书→呈签表→落盘）；任何"Agent 产出须人裁"的场景可复用此三层结构。

## changelog

- 2026-11-20：reflect 结晶（c-2026-00035）+人工精修（G2 遗留处置批，corr-2026-11-20-84）；⑦微调时窗候选按 Owner 裁并入本卡核心节（不另立卡）。