---
id: "corr-{yyyy}-{mm}-{dd}-{seq}"
layer: 1
type: correction
project: memory-lib
domain: []             # 被更正规格所属领域
maturity: evergreen    # L1 更正说明固定 evergreen（append-only 不可篡改）
target: ""             # 被更正的规格/卡 wikilink，如 [[SPEC-READ]]（必填）
corrections: []        # 回指被本说明取代的旧更正说明 id（可空）
created: {{date}}
last-touched: {{date}}
summary: ""            # ≤50 token 自包含：本次更正的核心
---

## 改了什么
<!-- 当前态相对旧态的差异，简明 -->

## 为什么改（原来错在哪）
<!-- 修订动机与旧态缺陷；这是后续 Agent 理解当前态的必要上下文 -->

## 依据
<!-- Owner 裁决 / log id / 实测锚点 / 磁盘证据 -->