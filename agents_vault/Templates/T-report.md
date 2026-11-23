---
id: "rep-{yyyy}-{mm}-{dd}-{seq}"
layer: 1
type: report
project: 
domain: []             # 本次变更涉及的领域
maturity: evergreen    # 申报报告一经提交即不可篡改（审计记录，同 L1 日志）
branch: ""             # 提交所在分支 contrib/{topic}-{date}
session-log: ""        # 关联会话日志 id [[log-...]]
touched: []            # 本次变更动过的文件 [[链接]]
doctor-result: ""      # 申报时 entry_doctor 自检结果摘要（pass/fail + 报告项）
created: {{date}}
last-touched: {{date}}
summary: ""            # ≤50 token 自包含：本次申报的核心变更
---

## 变更清单
<!-- 增/改/删了哪些文件 + 对应 [[wikilink]]，逐条列出 -->

## 依据来源
<!-- 每条变更的根据：log id / 外部来源 / 本机实测 -->

## doctor 自检
<!-- 跑 entry_doctor 的结果：通过项 / 报告项；离线降级时注明。检查项见 [[SPEC-GOVERN]] 的 doctor 闸门节 -->

## 冲突与存疑
<!-- 必须自列，不许空着不写：与既有卡的矛盾、不确定处，如实列、不洗白 -->

## 建议关系
<!-- 新卡/改卡的边（k/v/e）建议，供审核=并入 Agent 复核 -->

<!-- 本申报报告是贡献者的不可篡改声明，不含审核结论。审核=并入 Agent 的复核结论（并入/退回 + 理由 + merge commit）记在 merge 提交信息与审核方当日日志，不回改本文件（见 [[SPEC-GOVERN]] 申报报告节）。 -->
