---
id: "spec-write-001"
layer: 3
type: spec
spec-kind: write
trigger: always
project: memory-lib
domain: [memory-governance, knowledge-engineering]
maturity: growing
created: 2026-09-25
last-touched: 2026-11-23
summary: 写入规格书：定义"怎么存"——记入、归档、结晶、更正、敏感过滤、写权限矩阵（按 mode 分级）
---

# SPEC-WRITE 写入规格书 v0.9.4

## 职责边界

定义**写入侧**全部行为。读取侧见 [[SPEC-READ]]；变更如何经审查并入 main（角色/权限/申报报告/doctor 闸门）见 [[SPEC-GOVERN]]。

> **实现状态图例**：当前实现主体与未来目标主体分开陈述；凡标记为 `FUTURE / GHOST / NOT IMPLEMENTED` 的脚本、文件或 hook，仅表示规范中的目标组件，不代表当前磁盘上可调用的实现。

## 写权限矩阵

| 主体 | 04-Logs | 02-Cards | 03-Atlas | 01-Specs | 90-Meta |
|------|---------|----------|----------|----------|---------|
| Agent（`fast-iterate`，本库默认） | 仅追加新日志 | **可直接建/改卡**（留 provenance + 当日日志） | 可改自动生成区外内容 | 仅微词反馈流程（见下） | 分支上可改，经审核=并入 Agent + doctor 闸门并入 main（Git 兜底，A/D26）；MAP 仍禁改 |
| Agent（`careful`） | 仅追加新日志 | 禁止（走 reflect.py） | 禁止 | 仅微词反馈流程 | 分支上提案，经审核=并入 Agent 并入 main；MAP 禁改 |
| reflect.py --apply | 回填 reflect-to | 生成 seed 卡 | 禁止 | 禁止 | 禁止 |
| `atlas_fill.py --apply`（已落地，[[atlas_fill#VERSION|v0.1.0]]，corr-95；原 GHOST 标签滞留已清，corr-103） | 禁止 | 禁止 | 仅自动生成区 | 禁止 | 禁止（MAP.yaml 显式除外，仍 GHOST；本工具只做 SH 投影） |
| `build_graph.py` *(FUTURE / GHOST / NOT IMPLEMENTED)* | 禁止 | 禁止 | 禁止 | 禁止 | 仅内部缓存 |
| 人 | 任意 | 任意 | 任意 | 任意 | schema 手改，MAP 禁改 |

> mode 语义见 [[SPEC-READ#运行模式与认知姿态]]；无论何档，真安全红线（§安全 A）绝对生效。fast-iterate 放开 L2 直写的前提是 Git 兜底已张开（vault 已 git init）。变更并入 main 的治理流程（branch→申报报告→审核=并入 Agent→merge）见 [[SPEC-GOVERN]]；当前处过渡模式（直提 main + Git 兜底），角色实例化待定。

## 写入通道（Obsidian 运行时优先，D19/D23）

写入主走 **官方 `obsidian` CLI**（第一方、byte-exact）；App 未运行 / CLI 不可达时**显式降级**到离线原子文件写。Local REST API **不参与写入**（仅只读，见下）；插件 MCP 是**读取侧**主通道（见 [[SPEC-READ]]），不碰写入。

| 写入动作 | 官方 CLI（主） | 离线兜底（降级） |
|----------|---------------|-----------------|
| 新建卡 / 新日志 | `obsidian create path= content=` | 原子文件写（temp+os.replace） |
| 追加日志 | `obsidian append path= content=` | 文件追加 |
| 改属性（frontmatter） | `obsidian create path= content= overwrite`（整篇字节精确重写） | 文本替换 |

- **CLI 二进制须解析为全路径控制台启动器 `Obsidian.com`**：Windows `CreateProcess` 对裸名 `obsidian` 默认补 `.exe` 启动 225MB GUI 版，对 `property:set`/`search:context` 等冒号命名空间命令返回 rc=-1（假崩溃）；`append_log.py` 用 `shutil.which` 解析全路径、落 `.exe` 时优先取同名 `.com` 兄弟文件，见 [[PIT-obsidian裸名误调GUI版]]。
- CLI 写入须 Obsidian App 运行且**活动 vault 命中目标 vault**（`obsidian vault info=path` 守卫，不符即降级 offline）；不加 `open`/`newtab` 即不自动打开文件（**无 `silent` 参数**，旧版误记已撤销）。
- **content 转义契约**：CLI 把字面 `\n`/`\t` 反转义为真换行 / 制表符，故 `append_log.py` 把真换行编码为字面 `\n`（实测字节精确）；但内容含字面反斜杠+n/t（如 Windows 路径 `C:\temp`）会被误转义且无法规避 -> 检测到即**显式降级 offline**。
- **改属性不用 `property:set`**：它会把整个 frontmatter 规范化为 Obsidian block 风格 YAML（内联列表 -> block、剥引号），破坏本库 inline 约定（见 [[PIT-frontmatter重排为block风格]]）；改属性一律走 `create overwrite` 整篇字节精确重写。
- **写后回读校验**：CLI 写入后从磁盘回读、与意图内容逐字节比对，不符即 fail-loud（提示改用 `--channel offline`）。
- **禁用 `obsidian eval`/`command`/`delete`/`move`/`dev:*`**（见§安全 A）。Windows 消费 CLI 输出前设 `[Console]::OutputEncoding=[Text.Encoding]::UTF8`。

### 守卫式写入原语 `append_log.py`（批次4）

L1 记入与派生指针回填统一走 `90-Meta/scripts/append_log.py`（真值源=脚本卷 [[append_log]]，.py 无常驻文件、用时经 [[vol_extract]] 抽取；纯标准库、默认 dry-run、`--apply` 才写、原子写 temp+os.replace），把"append-only 守卫"落成可执行原语：
- **两动作**：`new-log`（安全追加新 L1 日志）/ `backfill`（向既有日志 frontmatter 的派生指针列表 reflect-to/corrections/superseded-by 追加一值，正文逐字节不动、幂等）。
- **四守卫**：防重复（新 id 不得已存在、目标文件不得已存在 -> 绝不覆盖历史）；append-only（backfill 只改 frontmatter 目标字段，校验正文不变）；**ifMatch 式防漂移**（离线用内容 sha256 复刻 Local REST API 的 `version` token：`--expect-hash` 不符则 exit 3 中止、文件不动）；**block-style fail-loud**（目标字段若被 Obsidian UI/REST 重排为 block 风格 YAML，离线 inline 写法会孤儿化其值 -> 检测到即 exit 3 拒绝、不静默丢数据，见 [[PIT-frontmatter重排为block风格]]）。
- **三通道**（`--channel`）：`cli`（**默认、写入主通道**——官方 `obsidian create`：new-log -> create、backfill -> create overwrite，均字节精确；CLI 不可用（启动器缺失 / App 不可达 / 活动 vault 不符 / 内容含字面 `\n`-`\t`）时**显式降级 offline**，写后回读逐字节校验）/ `offline`（原子文件写兜底，self-test 覆盖，CLI 降级目标，逐字节精确）/ `rest`（**可选、只读** Local REST API 适配器，第三方插件由用户自装、TTL 不代装不代跑；`--key-from-plugin` 运行时从插件 data.json in-memory 读 apiKey（该文件已 gitignore、绝不打印/落盘），或 `OLR_API_URL`/`OLR_API_KEY` 由用户 env 提供、禁写入任何层级；**仅 GET document-map 验连通 + 取 ifMatch `version` token**，`--apply` 明确拒绝——实测发现插件 frontmatter PATCH 会把整个 YAML 从 inline 重排为 block 风格并剥引号，破坏本库 inline 约定 + 离线 `parse_inline_list`（静默读空），见 [[PIT-frontmatter重排为block风格]]；绝不调用 POST/PATCH/delete/move/command_execute/eval）。
- 退出码：0 成功 / 2 工具错误 / 3 守卫拒绝（重复/漂移/append-only/block-style）。`--self-test` 离线自检 10 规则。
- **版本与输出流**（v0.3.0，2026-11-13）：工具持 `VERSION` 常量（与 entry_doctor / read_orchestrator / reflect 同惯例）；**全部输出（含 `vault not found` / `GUARD REJECTED` / `ERROR`）走 stdout**，与其余三个入口统一，调用方只需接一个流；退出码与四守卫语义不变。见 [[corr-2026-11-13-37]]。

## 记入协议（L1，会话结束时执行）

1. 每次有实质产出的会话追加一篇日志（模板 `Templates/T-log.md`），frontmatter 必填：id/project/session-task/touched/created；**经 `append_log.py new-log` 守卫式落盘**（防重复 id / 防覆盖 / dry-run diff，见§写入通道）；
   - **`touched` 语义**：本次会话**涉及**的 vault 文件——**读取 / 引用 / 修改均计入**（不限于实际改动），每项用 `[[链接]]` 表达。touched 记录 Agent 的**操作关联**（"本次涉及哪些文件"），其链接形式会被 Obsidian 链接引擎识别为图边（技术副作用），但 touched 本身**不等于** L2 卡的 semantic correspondence（带 `rel` 类型的语义边）。当前**允许 `touched: []`**（schema 无 minItems、L1 不判 orphan）；是否应要求 L1 最小关系数属未决（候选 C2，NOT OWNER DECISION）。
2. 正文四段固定：做了什么 / 结论与产出 / **存疑与风险（必须自列，不许空着不写）** / 候选结晶；「做了什么」段**首行固定 SH 身份行**（D34 留痕，v0.9.4）：`SH：身份=<维护者/消费者/联邦/审核者>；角色件=<role-ke-*>；通道=<MCP/REST/CLI/未涉库>`——一行非声明段，append-only 留案底供波级签收顺带抽查（可与会话工具调用记录对账），**不打回产出不惩罚**（[[SPEC-READ#入口与路由规则|§入口 6]] 触达惯例；行可伪造属 KNOWN-LEAKY，Owner 已显式接受，价值在归因非防伪造）；
3. **L1 日志正文 append-only 不可篡改**：历史日志的正文四段与 session-task/touched/结论一律不改，更正只能新开日志并在 frontmatter `corrections` 引用被更正日志 id；frontmatter 的描述字段（schema 合规回填）与派生指针（reflect-to/corrections）不在此限——回填 frontmatter 不算篡改正文；**L2 卡允许带卡内 changelog 的就地演进**（知识会生长，同 D12/D17 的 Git 兜底），重大改写记一笔 changelog 并更新 `last-touched`；
4. 敏感信息过滤清单（禁止写入任何层级）：密钥/token/密码/证书、个人身份证件信息、未脱敏的第三方隐私、可执行二进制内容。

## 全自动归档规则

- 每周反思编译（reflect.py --apply，真值源=脚本卷 [[reflect]]、用时抽取；Stage 2 前由 Agent 会话代行）：rollup 近 7 天日志的"候选结晶" → 生成 L2 种子卡（`maturity: seed`、`provenance: reflect`、来源引用 log id）自动入库；
- 回填原日志 `reflect-to` 字段；
- 日志 30 天后自然降权（检索时 recency 衰减处理），不删除、不移动。

## 结晶规则（seed 卡最低标准）

1. 有 `summary`（≤50 token 自包含定义）；
2. 至少一条边（k/v/e 任一），无边即孤岛，entry_doctor 报告；
3. `provenance` 标记来源（manual/reflect/feedback）；
4. 存疑内容结晶时必须在卡内"存疑"节保留原始不确定性，不得洗白为断言。
5. **结晶质量门**（内迁批 4 立条文，[[corr-2026-11-16-61]]；判据从批 1~2 已成熟的拓扑长出，非先验设计）：
   - **命名**：卡名 = 名词短语，≤10 字优先；技术术语保留英文原名；口语词（手感/踩坑/生草等）不入正式卡名；同范畴统一后缀。存量生草名 aliases 兜底，**不发起改名运动**；因其他原因修订时顺手规范化。治理族（constraint/reference/status）命名另一套规则：编号前缀 + 英文专名合法（DP-6 零改名判例），本命名节约束知识卡。
   - **分流**（抽象层就位后生效）：结晶前先查既有枢纽（`hub: true` 卡 + 治理原理节点）能否承载——能承载则**挂靠**（深化宿主章节 + implements 边，abstracts 互逆），不能则评估新建；近义兄弟卡须 `contrasts_with` 显式区分否则不新建（幂等守卫同款）。
   - **升迁联动**（B6）：被 N 卡 implements/grounds = 成熟证据类型之一；挂靠深化宿主后提示重评 maturity——提示不是自动闸门（F 原理），maturity 裁量仍归 Owner。
   - 新建卡须满足上列 1~4 既有质量门。机械检查（CARD_NAME WARN 级）暂缓：黑名单/长度线待实践样本校准，误报归 Owner 裁不硬拦（防 Goodhart）。

## 更正规则

分层处理：**L1 日志**正文 append-only 不可篡改——发现错误 → 新日志引用原 id 声明更正 → 原日志仅在 frontmatter `corrections` 回填指针（回填 frontmatter 不算篡改正文）；**L2 卡**允许就地演进——小修直接改正文 + 卡内 changelog，被整体取代时才用 `superseded-by` 指向新卡；**L3 规格**按微词反馈流程直接修订，**修订说明下沉至 `04-Logs/corrections/`（type:correction），规格正文只留当前态 + `[[corr-…]]` 指针，不再内嵌 changelog**（v0.7，Owner 2026-11-12）。三者统一由 Git 兜底回滚（D17）。

**更正说明件（type:correction）**：落 `04-Logs/corrections/`，模板 `Templates/T-correction.md`，正文三段（改了什么 / 为什么改（原来错在哪）/ 依据），frontmatter `target` 必填（被更正规格/卡）、`corrections` 回指被取代的旧说明 id。**不进自动装配、不结晶进 L2**（U-1）——只做「顺规格指针按需展开」的深度，其持久 provenance 语义不受日志 recency 衰减影响。与情景日志同属 L1 链层但语义为注脚，寄居情景层是务实复用（同 D21/D22）。

## 微词反馈流程（规格书自进化引擎，决策 D5/D12）

1. 用户向当时负责的 Agent 反馈微词（跨会话凭 log id 定位上下文）；
2. Agent 义务三连：
   a. 创建 feedback 卡（`type: feedback`，引用原 log/卡 id）；
   b. 判定是否构成 ⚠️点 → 生成 pitfall 卡（模板 `Templates/T-pitfall.md`）并设 `trigger`；
   c. 判定是否暴露规格书缺陷 → **直接修订** SPEC-READ/SPEC-WRITE/SPEC-GOVERN 对应分区，**修订说明下沉一件 correction**（`04-Logs/corrections/`，模板 T-correction，经 `append_log.py new-log` 守卫落盘）+ 规格「修订说明（已下沉）」节追加 `[[corr-…]]` 指针（Git 兜底回滚）；治理上线后走 [[SPEC-GOVERN]] 的 branch→申报报告→审核=并入 流程；
3. 修订在当日日志中高亮标记，供用户抽查复核。

## 安全

### A. 真安全红线（绝对，任何 mode 下不放宽）
1. 不安装/执行第三方 skills/插件包；引入外部代码前人工读全文；
2. 敏感信息（密钥/token/密码/证书/隐私）禁止写入任何层级（见记入协议过滤清单）；
3. vault 不接云端自动上传；语义索引仅本地 embedding（如启用）；
4. 网络调研只取文本，不下载执行、不盲跑网页命令；
5. 通道分级：官方 `obsidian` CLI 第一方（不触发第三方红线）；Local REST API/MCP 第三方执行体，仅用户审源信任后自装，Agent 不代装/代执行；`obsidian eval` 一律不用。

### B. 谨慎性约束（随 operating-mode 分级，D15）
1. 写入脚本默认只读、显式 `--apply`：`careful` 档 `--apply` 前输出完整变更预览并确认；`fast-iterate` 档下**可逆**写入可免确认、做完报告；
2. vault 纳入 Git 版本控制、`.obsidian/workspace.json` 与整个 `.obsidian/plugins/`（插件 `data.json` 含密钥、`main.js` 大二进制）入 .gitignore——Git 是所有直接修订的兜底，故 fast-iterate 下可放心直改（前提：已 git init）；插件密钥泄漏坑见 [[PIT-插件密钥入库泄漏]]。

## 修订说明（已下沉）

自 schema v0.7 / Owner 2026-11-12 起，规格修订说明不再内嵌于本文件，下沉至 `04-Logs/corrections/`（type:correction，按需顺指针展开，不进自动装配）。

本文件历史修订说明：[[corr-2026-09-25-10]], [[corr-2026-09-26-11]], [[corr-2026-09-26-12]], [[corr-2026-09-27-13]], [[corr-2026-09-27-14]], [[corr-2026-09-28-15]], [[corr-2026-09-28-16]], [[corr-2026-09-29-17]], [[corr-2026-09-30-18]], [[corr-2026-10-07-19]], [[corr-2026-11-12-36]], [[corr-2026-11-13-37]], [[corr-2026-11-16-52]], [[corr-2026-11-20-92]]（D34 记入协议第 2 条 SH 身份行）。

后续修订按 [[SPEC-WRITE#微词反馈流程（规格书自进化引擎，决策 D5/D12）|微词反馈流程]] 新增 correction 件并在此追加指针；Git 兜底回滚不变（D17）。
