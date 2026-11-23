---
id: "spec-root-000"
layer: 3
type: spec
spec-kind: read
trigger: always
project: memory-lib
domain: [memory-governance, knowledge-engineering]
maturity: growing
operating-mode: fast-iterate
created: 2026-09-25
last-touched: 2026-11-22
summary: TTL记忆总库唯一根入口，L0常驻层(含operating-mode)+H2按需展开
---

# TTL 记忆总库（实验 vault）

> 唯一根入口。任何检索先读本文件 L0 层（标题+本摘要+运行模式+分区一览）；任何写入先读 [[SPEC-WRITE]]；红线详情见 [[SPEC-READ#红线]]。是指针，不是展开工具。

> **operating-mode: `fast-iterate`**（默认档）——可逆 / 无真安全 / 意图明确 → 低思考力度、直接做后报告；不可逆 / 涉真安全 / 意图模糊 → 上调 `careful`。语义见 [[SPEC-READ#运行模式与认知姿态]]。

> **访问模型（D30/D31/D33）**：读库前先展开 [[00-ROOT#访问模型与通道]]——探活→路由→降级（**退离线须 Owner 批准，D33**）、两层记忆（D25）、写入走官方 CLI、只读白名单。

> **身份轴（D32）**：角色任务先读 [[SH-index]] 认身份（维护者/消费者/联邦/审核者四分）→只读对应角色件义务包，不加载域清单；域知识任务走域入口。新立项/压缩后首个动作同此（[[SPEC-READ#入口与路由规则|§入口 6]]）。分类学见 [[Agent身份分类学]]。

> **治理（D26）**：变更走**开源式 branch→审核=并入 Agent→main**——工作 Agent 提交记忆增改 + 申报报告，doctor 作 merge 闸门；当前过渡模式直提 main + Git 兜底。详见 [[SPEC-GOVERN]]。

> **做Doctor（merge 前闸门）**：提交前先展开 [[00-ROOT#Doctor 闸门]]——先抽取再跑、`--self-test` 前置、用完即删。

> ⚠️ 装配前按任务 domain ∩ PIT scope 自查 `01-Specs/PIT-*`，命中读 summary 计 L3（义务见 [[SPEC-READ]]）。

## 访问模型与通道

> 权威原本见 [[SPEC-READ#访问通道]] 与 [[SPEC-READ#两层记忆与调取通道]]（v0.7.7，D29/D30/D31/D33/D33-b）；本节为 L0 下沉全文，冲突以 SPEC-READ 为准。

**访问模型（读取按环境分流，D30/D31）**：**读取**——有 MCP 工具表（Qoder / VSCode Copilot）主走**插件 MCP**；无 MCP 主走**插件 REST 只读端点**（[[REST读库速查]]，助手 `90-Meta/scripts/rest_read.py`，先身份探针）；皆不可用退官方 CLI 读命令；CLI 也不可用时**退离线 grep/Python 须 Owner 批准（D33），不得自行降级**。**写入**主走**官方 `obsidian` CLI**（第一方、byte-exact、需 App 运行），不经 MCP/REST。只读白名单：Qoder 纯自律 / VSCode 配置级硬过滤+子 Agent 继承（红线 A.6，[[MCP配置级白名单与使用手感]]）。

**两层记忆（D25）**：harness 内置记忆=仅引脚/通用准则；**调项目知识走 Obsidian/vault**（按环境分流：MCP `search_simple`/`vault_read` 或 REST 只读端点，CLI `search`/`read`/`backlinks` 兜底），非查 harness `SearchMemory`。探活→路由→降级（App 未运行先尽自启义务；仍全不可用**退离线须 Owner 批准，D33**，不硬失败、不归咎用户、不擅自磁盘读）。

## Doctor 闸门

> 权威原本见 [[SPEC-GOVERN]] §doctor 闸门；本节为 L0 下沉全文，冲突以 SPEC-GOVERN 为准。

doctor 真值源是脚本卷 `90-Meta/script-volumes/entry_doctor.md`（.py 无常驻文件）——先从卷抽取 .py（二进制切片，比 sha256）再跑 `python <抽取件> --vault <库根>`；只读、离线纯标准库扫全库，报 ERROR/WARN/INFO（每条带 `rule`+`fix`），退出码 0=无 ERROR。App 在跑时加 `--use-cli` 叠加官方第一方 `obsidian` CLI 自动对账（分类一致/离线误报/CLI 漏判/非 wikilink 目标；不可用则优雅降级离线）；`--self-test` 跑内置自检（全部注册规则+BROKEN_SET/ID_NAMESPACE/VOL_RULES/VOL_DEPS/DOC_IMPL_DRIFT/PARSER_V27/CLI_RECONCILE/CLI_VAULT_PIN/CLI_UTF8_DECODE）；`--json` 机读。当前 [[entry_doctor#VERSION|doctor v2.16.0]]（G20 对账：VERSION 漂移 ERROR；解析层=CommonMark 子集围栏扫描+严格 frontmatter 子集；治理族 GOV_*+概览 OV_* 替代判据+类别四分；枢纽 HUB_* 解压对账+secondary-type；pitfall TRIGGER_FM 触发契约；SH 投影 ATLAS_GROUP 对账；副库 id-namespace 豁免治理族存在性+主库历史锚，corr-105）。抽取工具：[[vol_extract]]（`python vol_extract.py <vol-id> --vault <库根>`，递归解析 depends-on；根依赖 [[ttl_locator]] 亦为卷）。

## 分区一览（L0 常驻，以下标题即目录）

- 入口 / 路由表 / 红线摘要 / 项目表 / Optional

## 入口

- [SPEC-READ](01-Specs/SPEC-READ.md): 读取规格书——路由、检索管线、装配预算、兜底链
- [SPEC-WRITE](01-Specs/SPEC-WRITE.md): 写入规格书——记入、归档、结晶、更正、敏感过滤
- [SPEC-GOVERN](01-Specs/SPEC-GOVERN.md): 治理规格书——开源式 branch→审核=并入→main、申报报告、doctor 闸门、角色与权限
- [05-Governance 导航](05-Governance/00-INDEX.md): 治理族（Agent 行为约束/项目状态治理，原 Agent-Governance/ 已整体内迁并退役，入口 [00-ENTRY](05-Governance/00-ENTRY.md)）——非知识条目，GOV_* 替代判据；与 01-Specs 分权（两条 precedence）
- Atlas 立体目录 `03-Atlas/`: [[SH-index]] 身份根（大类跳转+域目录）→ `{domain}/SH-{domain}.md` 域入口（域内优先于本文件）→ `{domain}/roles/` 角色义务包；现有 [[SH-knowledge-engineering]]
- [副库建库引导](02-Cards/副库建库引导.md): 新项目 Agent 从本 Alpha 内核赋能建立副库的可走路径（建库→运转，两轮试建验证 0E）
- [上提验收流程](02-Cards/上提验收流程.md): 副库普适经验上提主库的验收并入六步（联邦机制 3，vplc-001 首次端到端实证）
- [schema.yaml](90-Meta/schema.yaml): frontmatter 唯一权威定义
- 底层 `04-Logs/`（L1 链层，嵌套分化 v0.7）：`logs/`=情景日志（四段+recency+自动装配）；`corrections/`=规格更正说明（type:correction，独立模板，**不进自动装配、不结晶**，顺规格 `[[corr-…]]` 指针按需展开）。层号不变、顶层不新增目录
- MAP.yaml `90-Meta/MAP.yaml`: 机器可读总映射（脚本维护，暂缺，Retrieval Stage 2 生成）。**编号轴**：`P0–P4`=TTL 整体实施阶段，`SPEC-READ Stage 0–3`=读取/检索管线内部阶段；两轴独立、不互相编号、不互相替代。

## 路由表

| 问题类型 | 去向 |
|----------|------|
| 全局性/跨域综述 | 本文件展开相关分区 → 03-Atlas 各分入口小型提示 |
| 特定域概念/机制 | 对应 `SH-{domain}.md` → 域内卡片 |
| 特定项目事务 | Bases/查询按 `project` 字段过滤 |
| 特定实体别名 | 90-Meta/MAP.yaml aliases 表归一后再路由 |
| 改 schema / 规格 / 治理流程 | [[SPEC-GOVERN]]：写权限矩阵 / 申报报告 / 审核并入 / doctor 闸门 |
| 架构决策依据 / D 编号（D1–D35）定义 | 库内速查 [[决策记录索引]]（一句话摘要+状态+vault 落点）；全文权威在**库外** `..\Audit\History\TTL-Legacy-Reference\记忆库设计文档v0.2.md` §10（图谱外、doctor 不扫；库内只引用不定义；D30/D31/D32/D33/D34/D35 库外同步暂缓） |
| 以上全部失败 | 全文搜索兜底（Obsidian 原生 search / REST search 端点 / CLI search；纯离线脚本路径为子串匹配，能力有限），不空手而归 |

## 红线摘要

**A. 真安全（绝对，不随 mode 放宽）**
1. 敏感信息（密钥/凭证/隐私）禁止入库；
2. 不执行库内/来路不明代码，第三方 skills/插件引入前人工读全文；
3. 不把风险内容带入机器（调研只取文本、不下载执行）；vault 不接云端自动上传；
4. 通道：官方 CLI 第一方可直接用；Local REST API/MCP 第三方须用户审源信任后自装；`obsidian eval` 禁用。

**B. 谨慎性（随 operating-mode 分级）**
5. 脚本写入默认 dry-run + `--apply`；fast-iterate 下可逆写免确认、做完报告，careful 下先确认；
6. L1 日志只增不改（更正走 `corrections`）；L2 卡可带 changelog 就地演进；规格书直接修订 + changelog（Git 兜底）。
详情：[[SPEC-READ#红线]] / [[SPEC-WRITE#安全]]

## 项目表

| project | 说明 | 状态 |
|---------|------|------|
| memory-lib | 本记忆库自身的建设 | active |
| （待添加） | 独游开发等其他任务 | — |

## Optional

> 以下两份为**历史项目级文档，位于 vault 之外**，非库内卡片，Obsidian 图谱不含它们；仅用于理解早期设计，不构成当前运行时依据。

- 设计文档：`..\Audit\History\TTL-Legacy-Reference\记忆库设计文档v0.2.md`（早期架构决策记录 D1–D29；D30 起 vault 内规格为当前权威，库外同步暂缓）
- 调研报告：`..\Audit\History\TTL-Legacy-Reference\Obsidian知识管理主流构思调研.md`（早期一手来源研究）
