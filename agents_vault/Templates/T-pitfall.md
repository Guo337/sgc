---
id: "pit-{yyyy}-{seq}"
layer: 3
type: pitfall
trigger: on-domain     # always|on-domain|on-file-pattern|on-project
scope: ""              # trigger 的匹配范围（域名/文件模式/项目名）
project: 
domain: []
maturity: seed
provenance: feedback   # ⚠️点通常源自微词反馈
created: {{date}}
last-touched: {{date}}
summary: ""            # ≤50 token：一句话说明"什么情况下会踩什么坑"
e: []                  # 至少一条边，通常 grounds 指向相关规格分区或卡
---

# ⚠️ {{title}}

## Bug 类型
<!-- 哪一类错误 -->

## 根因
<!-- 为什么发生 -->

## 修复模式
<!-- 怎么修 -->

## 适用边界
<!-- 什么情况下此教训有效/无效 -->

## 来源
<!-- 引用的 feedback 卡 / log id -->
