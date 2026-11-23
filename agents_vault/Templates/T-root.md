---
id: "spec-root-000"
layer: 3
type: spec
spec-kind: read
trigger: always
project: {{project}}
domain: [memory-governance, knowledge-engineering]
maturity: growing
operating-mode: fast-iterate
id-namespace: {{slug}}
created: {{date}}
last-touched: {{date}}
summary: "{{project}}副库根入口，L0常驻层+H2按需展开（由TTL主库Alpha内核赋能建立）"
---

# {{project}} 记忆副库

> 唯一根入口。任何检索先读本文件 L0 层；任何写入先读 [[SPEC-WRITE]]；红线见 [[SPEC-READ#红线]]。是指针，不是展开工具。
> 本副库由 TTL 主库（Alpha 内核）赋能建立，骨架与主库同源，项目内容自填。

> **operating-mode: `fast-iterate`**——可逆/无真安全/意图明确→直接做后报告；不可逆/涉真安全/意图模糊→上调 careful。语义见 [[SPEC-READ#运行模式与认知姿态]]。

> **边界（TTL 存 Agent 经验，不存外部知识本体）**：本库只结晶本项目 Agent 做事的经验（踩坑/方法/判断）；外部文献/资料本体留库外工具，库内只留指针。

> **id 命名空间（`id-namespace: {{slug}}`，Jira/GitHub 同构）**：本库**新建**条目 id 必须带本库前缀——L2/L3 用 `{kind}-{{slug}}-{seq}`（如 `c-{{slug}}-00001`），L1 用 `{kind}-{{slug}}-{yyyy}-{mm}-{dd}-{seq}`。**赋能复制来的上游条目保留原 id 并标 `provenance: copied`**（上游身份引用，豁免前缀校验）；**不许续主库号段**（会与主库未来新卡撞号）。上提主库时由验收方换主库号。doctor 按 ID_NAMESPACE 校验。

## 分区一览（L0 常驻）

- 入口 / 路由表 / 红线摘要 / 项目表

## 入口

- [SPEC-READ](01-Specs/SPEC-READ.md): 读取规格书
- [SPEC-WRITE](01-Specs/SPEC-WRITE.md): 写入规格书
- [SPEC-GOVERN](01-Specs/SPEC-GOVERN.md): 治理规格书
- Atlas 分入口 `03-Atlas/{domain}/`: 域入口按域子目录组织，现有 [[SH-knowledge-engineering]]
- [schema.yaml](90-Meta/schema.yaml): frontmatter 唯一权威定义
- 底层 `04-Logs/`：`logs/`=情景日志；`corrections/`=规格更正说明

## 路由表

| 问题类型 | 去向 |
|----------|------|
| 全局性/跨域综述 | 本文件展开 → 03-Atlas 分入口 |
| 特定域概念/机制 | 对应 SH-{domain}.md → 域内卡片 |
| 改 schema/规格/治理 | [[SPEC-GOVERN]] |
| 以上全部失败 | 全文搜索兜底，永不空手 |

## 红线摘要

1. 敏感信息禁止入库；2. 不执行来路不明代码；3. 不接云端自动上传；4. obsidian eval 禁用。
详情：[[SPEC-READ#红线]] / [[SPEC-WRITE#安全]]

## 项目表

| project | 说明 | 状态 |
|---------|------|------|
| {{project}} | （本项目说明） | active |