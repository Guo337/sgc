---
id: "rep-ansys-2026-11-23-01"
layer: 1
type: report
project: ansys-automation
domain: [knowledge-engineering, memory-governance]
maturity: evergreen
branch: "contrib/vol-integrity-2026-11-23"
session-log: "log-ansys-2026-11-23-05（ansys-automation 副库）"
touched: ["[[c-ansys-00004-脚本卷字节完整性纪律]]"]
doctor-result: "副库 entry_doctor v2.15.0：92 文件 0 ERROR / 0 WARN / 0 INFO（离线断链模式；交件前末次自检）"
created: 2026-11-23
last-touched: 2026-11-23
summary: "上提 1 张普适卡候选：脚本卷字节完整性纪律（sha256 锚定件批量处理豁免+Git EOL 防护）；关联缺陷反馈 issue #1"
---

## 变更清单

- 新增交件：`90-Meta/inbox/ansys-2026-11-23/c-ansys-00004-脚本卷字节完整性纪律.md`（1 张卡，保留副库 id，source 指针指副库路径与源日志）。
- 本报告 REPORT.md。
- 只写本 inbox 子目录，未碰主库其他文件。

## 依据来源

- 副库建库实测（2026-11-23）：批量裁剪触发 5 个 VOL_SHA256 ERROR → 字节原样恢复后归零；全过程见副库 log-2026-11-14-01（建库日志）与 log-ansys-2026-11-23-04。
- 缺陷反馈已先行提交 issue #1（https://github.com/Guo337/sgc/issues/1），本交件是其普适卡候选部分的标准化补交（issue 渠道不便携带 inbox 目录结构）。

## 普适性论证

- **换项目检验**：纪律对象是"以内容哈希为完整性锚的文件"的通用处理规则（批量文本处理豁免 + 版本管理器 EOL 豁免），与 Ansys 业务零耦合；任何建副库的项目在 Windows/Git 环境都会遇到。
- **主题归属检验**（corr-98 判据）：去掉项目专名后，主题是"知识库完整性锚如何不被日常操作破坏"——属记忆治理/知识工程论域，非外部工具技能。

## doctor 自检

- 副库全库 92 文件 0E/0W/0I（离线断链模式，Obsidian CLI 未装故未跑 --use-cli 对账，如实注明）。
- 交件卡 frontmatter 按 schema 扁平子集自查：必填齐、e 边 2 条（targets 均为主库既有卡，主库内可解析）、summary ≤50 token、存疑节未洗白。

## 冲突与存疑

- **与引导卡关系**：补充非冲突——引导卡坑清单现无卷裁剪/EOL 两条（issue #1 问题 3）。维护者可选择独立并入或合并进引导卡坑清单，本产出者不预断。
- **争议分离**：副库原卡"核心"第 4 条含治理族复制口径争议实例（GOV_SINGLETON 与 federation:never 张力），已按 issue #1 拆出——争议走 issue 裁决，本卡只留字节完整性技术纪律。
- **单次实证**：EOL 碎锚在 Windows autocrlf=true 实测；其他 Git 配置组合未扫描。
- **inbox 索引污染风险**（上提流程已知项）：本交件经 GitHub PR 递交，主库合并后请按流程归档移出 Obsidian 索引。

## 建议关系

- `e: implements → [[副库建库引导]] (0.7)`：本卡是建库第 2/4 步的完整性防护细则。
- `e: co_occurs → [[entry_doctor校验清单与自检法]] (0.5)`：VOL_SHA256 判据的触发与防护。
- 若并入：建议同时在引导卡坑清单加指针（由验收方裁）。

<!-- 本申报报告是贡献者的不可篡改声明，不含审核结论。审核结论记 merge 提交与审核方日志。 -->