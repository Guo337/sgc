# 上提收件箱（inbox）

> 副库向主库上提普适经验的**收件处**。NON-AUTHORITY 工作区，不是库内正式卡片层。
> 副库 Agent 把「普适卡 + 申报报告」导出到此目录；主库 Agent 在此验收，过则并入 02-Cards（分配主库 id），不过则退回。

## 目录约定

```
90-Meta/inbox/
  README.md                  ← 本说明
  <submission-id>/           ← 一次上提一个子目录
    card-<n>.md              ← 副库普适卡原文（保留副库 id，验收时换主库 id）
    REPORT.md                ← 申报报告（T-report 结构 + 跨库来源）
```

## 验收标准（主库 Agent 用，对齐 SPEC-GOVERN + 联邦收录边界）

1. **普适性**（核心）：换个项目还管用吗？只收「一般机制」，不收「本项目专属事实」。
2. **结构合规**：frontmatter 合规、有边、summary≤50token、存疑不洗白（doctor 规则）。
3. **来源可核**：grounds 指回副库实例（跨库指针），provenance 记副库来源。
4. **不冲突**：与主库既有卡无矛盾/重复（重复则建议 supersedes 或合并）。
5. **id 不撞**：并入时换主库 id 序列（**以磁盘实测最大号为准**，勿信本文件写死的数字——卡号随库增长），原副库 id 记入 provenance。

## 角色分离

产出者（副库 Agent）≠ 验收者（主库 Agent）。验收结论记主库日志 + merge 提交，不回改副库申报报告。重大/普适性争议归 Owner 终审。