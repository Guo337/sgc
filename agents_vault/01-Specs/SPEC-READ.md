---
id: "spec-read-001"
layer: 3
type: spec
spec-kind: read
trigger: always
project: memory-lib
domain: [memory-governance, retrieval]
maturity: growing
created: 2026-09-25
last-touched: 2026-11-24
summary: 读取规格书：定义"怎么找"——路由、检索管线、装配预算、兜底链、运行模式、红线
---

# SPEC-READ 读取规格书 v0.7.8

## 职责边界

本规格书定义**读取侧**全部行为：入口路由、检索管线、上下文装配预算、兜底链、运行模式、红线。写入侧见 [[SPEC-WRITE]]。二者独立演化，微词反馈按问题性质路由到对应规格书修订。

> **实现状态图例**：当前实现主体与未来目标主体分开陈述；凡标记为 `FUTURE / GHOST / NOT IMPLEMENTED` 的脚本、文件或 hook，仅表示规范中的目标组件，不代表当前磁盘上可调用的实现。

## 运行模式与认知姿态（operating-mode，v0.3 新增，D14/D15）

Agent 的"思考力度"与"确认模式"不是常量，而是由 `operating-mode` + 条目 `risk` 派生的两个旋钮。默认档位在 [[00-ROOT]] 的 L0 声明，可按 project/session 覆盖。

| operating-mode | 适用 | 思考力度 | 确认模式 |
|----------------|------|----------|----------|
| `fast-iterate`（本库默认） | 可逆、无真安全、意图明确的实验/迭代 | 低——按最合理读法直接推进 | 先做后报告（可逆操作免 dry-run 确认） |
| `careful` | 生产、不可逆、涉真安全、意图模糊 | 高——先厘清再动 | 先确认后执行 |

**上调判据**：可逆性 + 是否触及真安全红线 + 项目模式 + 意图明确度。任一命中"不可逆 / 涉真安全 / 意图模糊" → 上调 `careful`，即使全局是 fast-iterate；条目可用 frontmatter `risk: high` 就地强制上调（见 schema）。

**关键区分**：真安全红线（§红线 A）绝对、永不因 mode 放宽；谨慎性约束（§红线 B）才随 mode 分级。混淆二者 = 在零风险的可逆操作上过度设防。

## 入口与路由规则

1. 会话启动：读 [[00-ROOT]] 的 L0 层（H1 + blockquote + 分区标题），预算 ~500 token；**L0 当前构成（2026-11-18 指针化后）**：H1 + 根入口摘要 + operating-mode + 访问模型指针行 + 治理行 + Doctor 指针行 + ⚠️PIT 自查行 + 分区一览；访问模型/两层记忆/Doctor 全文已下沉 ROOT H2（[[00-ROOT#访问模型与通道]] / [[00-ROOT#Doctor 闸门]]），L0 只留祈使指针；
2. 按 ROOT 路由表判定问题类型 → 展开对应 H2 分区（`[[00-ROOT#分区]]` 锚点方式，禁止全文内嵌）；
3. 进入某域后：该域 `03-Atlas/{domain}/SH-{domain}.md` **优先于** ROOT 同名分区（子路径优先原则，采 llms.txt v2；域入口按域子目录组织，name_index 按 basename 解析路径无关）；
4. 分入口小型提示（首段 50–100 token）决定是否下钻到卡片；
5. 查询实体先经 `90-Meta/MAP.yaml` aliases 表归一（同义归一在映射层解决）；MAP 未生成前（Stage 0）以卡片原生 `aliases` 字段比对（`[[别名]]` 由 Obsidian 链接引擎自动解析）。
6. **新立项 / 压缩后触发**（Owner 2026-11-20 裁，corr-2026-11-20-87；19:02 扩压缩触发，corr-2026-11-20-90）：每次新立项（新波/新刀/新任务方向）开工前，**以及每次上下文压缩后的首个动作**，先走 [[SH-index]] 认身份（维护者/消费者/联邦/审核者）→读对应角色件义务包→认涉及域的 SH 域入口结构，再展开任务。日常已知目标的直读不强制走 SH（路由为"不知在哪"服务）；新立项与压缩后是"先验最不可靠"的时刻——压缩摘要是有损转述，凭摘要里的旧路径/旧义务先验定位=漂移温床（实证：2026-11-20 数轮不用 REST 的直接原因均为编排方压缩后跳过路由），SH 路由价值最高（D32 身份轴抗压缩初衷）。
   **触达惯例（D34，Owner 2026-11-20 20:21 裁「让 SH 被读到」）**：SH 本身是有效载荷——读到即 REST/诚实/不越权义务进视野，问题是"没读"不是"读了还犯"，故只配提醒+留痕，不设验证闸/声明核对/两段式（对低成本动作配高成本脚手架=本末倒置）。三落点：①**任务书首行**（标题下第一行）固定 SH 指针「先走 [[SH-index]] 认身份→角色件→域 SH，再开工（§入口 6）」——只指权威不手抄通道细节（[[不越权纪律]] 律 1）；②**轮次规划文件首行** blockquote 同款「接续/压缩后首个动作先走 SH」指针（搭"压缩后必读规划"必经路径便车）；③**STATUS「Next task」行首**同款短指针（入口序列必读位）。三处均为一句指针非平行文本展开。**第四落点（触发端，v0.7.8/corr-106，Owner 2026-11-24 批准）**：仓库根 `AGENTS.md` 入口同款指针——宿主会话启动即注入、压缩后仍在，恰补「库内提示压缩后不可达」的结构缺口（corr-99 登记，外部评审 issue #7 供解）；不是 hook（D32 否决不变），只是把指针放到压缩不可达面之外的既有入口文件。**留痕**：L1 日志「做了什么」段首行固定「SH：身份=…；角色件=…；通道=…」（[[SPEC-WRITE#记入协议（L1，会话结束时执行）|记入协议]]），波级签收顺带抽查，不打回不惩罚。**KNOWN-LEAKY（E 原理，Owner 显式接受）**：编排方侧无机械解（D32 否决 hook、session_start 照不到压缩后），提醒+留痕只降概率不保证；残余属小概率意外非结构性漂移（Owner 20:29 定性）。**注（corr-99/106）**：压缩场景的残余经 Owner 2026-11-22 归因修正为**结构性**（触发端在库内、压缩后不可达），非单纯概率意外；第四落点（AGENTS.md）即针对该结构缺口的修复，宿主不注入 AGENTS.md 的环境仍属 KNOWN-LEAKY。跳过 SH=约束逸出，违规定性挂 [[不越权纪律]] 实例节。

## 两层记忆与调取通道（D25）

调取前先分清 Agent 面前的**两层"记忆"**，否则会用错通道：

| 层 | 载体 | 内容 | 何时调 |
|----|------|------|--------|
| harness 内置记忆 | Agent 内存（`SearchMemory`） | **仅引脚 + 跨会话通用准则**（D18：非项目知识本体） | 取引脚 / 用户偏好 / 通用行为准则 |
| vault 知识本体 | Obsidian 库（唯一权威） | 项目全部领域知识 / 结论 / 经验 | **调项目知识 → 走下方 Obsidian 通道** |

**操作规则**：凡"调取项目知识 / 记忆"**默认指走 Obsidian 通道从 vault 取**（按环境分流：有 MCP 走插件 MCP `search_simple`/`vault_read`，无 MCP 走插件 REST 只读端点，官方 CLI `search`/`read`/`backlinks` 与离线兜底次之，D30），不是查 harness `SearchMemory`。混淆两层＝检索侧层级误判，见 [[PIT-记忆调取层级混淆]]（与删除侧 [[PIT-私有记忆与引脚混淆]] 成对）。

**调取前置协议（探活 → 路由 → 降级，环境感知 D30）**：
1. **探活**：先看会话工具表有无 obsidian MCP——**工具表存在 ≠ 可用，须一次实调探活**（如 `vault_read` 取 00-ROOT），实调成功才算 MCP 主通道可用；实调失败（如 harness MCP 客户端 transport failure，即使服务端 REST 200）则降 REST。无 MCP 工具表则探插件 REST（`rest_read.py probe` 或 `GET /`，含身份探针）；REST 也不可达则 `Get-Process Obsidian` + `obsidian version` 判断 CLI；
2. **路由**：MCP 可用 → 走插件 MCP；MCP 不可用但 REST 可达 → 走插件 REST 只读端点（[[REST读库速查]]）；皆不可用但 App 活 → 退官方 CLI；
3. **降级（D33，Owner 2026-11-20 裁：离线非法，须 Owner 批准）**：MCP/REST/CLI 全不可用（已尽自启义务仍不可达）→ **不得自行退离线**；须向 Owner 呈报通道不可用证据（探活/自启结果），**经 Owner 批准后才许退离线兜底**（grep/Python 扫 .md）。未获批准前停在该步、如实报告，不硬失败、不归咎用户（App/插件开关是运行时状态，非用户过错），但**绝不擅自磁盘读知识**。
   - **次级批准条款（D33-b，Owner 2026-11-22 裁设）**：非交互子 Agent（无 Owner 交互回路）执行**只读审计任务**时，**Reader 权限**（略次于 Owner，由审核者角色 [[role-ke-auditor]] 承载）可代行批准只读离线降级，条件四条全满足：①任务纯只读（无任何写入）②三通道探活+自启义务已尽③确无 Owner 交互回路④批准决定与逸出如实留痕（报告/日志）。**Reader 权限仅限只读离线降级一事**：不可批准写入、不可修改库、不可代行 Owner 终审/merge 权。设立依据：独立架构评审（2026-11-22）实证 D33 原条款假设随时在线的 Owner，非交互宿主下合规路径不存在（只读审计被卡死），与智能体无关化宣称张力；Reader 补此裂缝，不放松写入侧。
4. **子 Agent 冷启动**：全新上下文无对话记忆，须靠库文本发现读法——读 [[00-ROOT]] L0 访问模型行 → 按环境选 MCP/REST → 展开 [[REST读库速查]] 或 MCP 能力表。三路冗余发现见 [[冷启动可发现性三路径]]。
5. **按需自启 App（Owner 授权，仅限 Obsidian + 本目录，D31）**：REST/MCP 需 App 运行而 App 关着时，Agent **应**自启 Obsidian（本机 `Obsidian.exe`，`Start-Process` 拉起；`.com` 是 CLI 入口非 GUI 启动器，Start-Process 下不生效）并探活（含身份探针确认是本库），用完按需留/关；不替用户改其他运行态。**探活发现 App 关时，自启是授权内的首选动作，不应直接退离线**（退离线须先尽自启义务，corr-2026-11-15-44；自启后仍全不可用，退离线另须 Owner 批准，D33）。

**影子治理**：harness 可能**自动生成**项目知识影子记忆（即便 Agent 自律不存），违反 D18；发现即按"正确版入库 vault → 迁移后删内存影子（删前确认）"清理。故 D18"vault 唯一权威"不完全在 Agent 掌控内，需周期核对内存。

## 访问通道（读取按环境分流：MCP / REST / CLI / 离线，写入走官方 CLI，D19/D23/D24/D30）

**读取主通道按运行环境分流（D30）**——MCP 与 REST 是**同一 Local REST API 插件的两个面**（第三方增强适配器，已由用户人工审源信任后自装、仅只读接入），读的是同一个库、同一套语义：

- **有 MCP 工具表的环境**（如 Qoder IDE / VSCode Copilot，插件 MCP 已注册）：主走 **插件 MCP**（`search_simple`/`vault_read` 等 6 只读工具，红线 A.6；本机明文 HTTP `127.0.0.1:27123/mcp/` 实测跑通、无 401）。**MCP 注册落点（D31）**：Copilot harness 原生读 `~/.copilot/mcp-config.json`（`mcpServers` 格式）或工作区根 `.mcp.json`，**不直接读** `.vscode/mcp.json` / profile `mcp.json`（旧 Local harness `servers` 格式）；密钥用 `${ENV}` 引用不落字面。配置级工具白名单见红线 A.6 与 [[MCP配置级白名单与使用手感]]。
- **无 MCP 工具表的环境**（如 VSCode/Copilot 全新子 Agent，工具表里没有 obsidian MCP）：主走 **插件裸 HTTP REST 只读端点**（`GET /vault/{file}`、`POST /search/simple/` 等，见 [[REST读库速查]]；现成助手 `90-Meta/scripts/rest_read.py`，内置身份探针+只读白名单+降级）。**不得**因 MCP 缺席就硬失败或盲扫——REST 是此环境的读主通道。
- **MCP 与 REST 皆不可用**（App/插件未运行）：退**官方 `obsidian` CLI** 读命令（第一方，IPC 连 App）；CLI 也不可用时**不得自行退离线**，须呈报 Owner 批准后才许 Python/grep 离线兜底（D33）。

> **降级前置义务（v0.7.2 corr-43 探活前置；v0.7.5 D33 收紧：离线须 Owner 批准）**：退离线兜底前**必须先探活**——查 Obsidian App 是否在跑（`obsidian version` / REST `127.0.0.1:27123` / MCP 工具表），**证明主通道确实不可用**；探活+自启后仍全不可用，**还须向 Owner 呈报并获批准**才许退离线（D33，Owner 2026-11-20 裁「现在离线非法，必须得到 Owner 批准」）。**不探活直接默认离线 = 读取通道漂移；探活后未经 Owner 批准自行离线 = 越权降级**，两者皆非合规（活体实证：编排方与项目 Agent 均不探活就磁盘读，浪费 SH 路由/反链/对账）。「永不空手/不硬失败」指**通道确不可用且获 Owner 批准离线**时不卡死，**不是**「离线永远合规」的借口，**更不是**「Agent 可自行决定离线」的授权。**库只要被 Obsidian 管辖即被索引**——含工作区子目录下的副库（实测 project_researching vault 索引其 memory-vault 子目录，CLI 能答反链）；故副库读写同样应优先链接通道，离线是例外不是默认、且须 Owner 批准。doctor 对已索引库默认 `--use-cli` 对账。
>
> **引导期豁免（鸡生蛋，corr-2026-11-15-45）**：冷启动须先读入口规格（AGENTS.md / ROOT / SPEC-READ）才能发现读法——**发现读法本身的引导期读取可磁盘读**（此时尚未探活、不知通道用法）；**探活之后**的知识读取必须走链接通道。子 Agent 实测此边界合理（引导期磁盘读规格，探活后 REST 读卡 + CLI 反链）。

**写入不走 MCP 也不走 REST**，一律走官方 CLI（D23，byte-exact）或离线降级——见 [[SPEC-WRITE]]（REST 写端点会重排 frontmatter YAML，已禁用）。

| 读取能力 | 插件 MCP（有 MCP 环境主） | 插件 REST（无 MCP 环境主） | 官方 CLI（兜底） | 离线兜底 |
|----------|---------------|---------------|----------------|----------|
| 读卡/规格 | `vault_read` | `GET /vault/{file}` | `obsidian read file=` | 读文件 |
| 全文/属性检索 | `search_simple`/`search_query` | `POST /search/simple/?query=` / `POST /search/`(JsonLogic) | `obsidian search query=` | ripgrep/BM25 |
| 结构/版本 | `vault_get_document_map`（标题树+块ID+version token） | `GET /vault/{file}` Accept document-map+json | — | 解析 |
| 列目录/标签 | `vault_list`/`tag_list` | `GET /vault/{dir}/` / `GET /tags/` | — | glob |
| **反链（入边）** | `vault_read` 的 `backlinks`（原生、App 实时索引） | **无独立端点**（退 CLI/离线） | `obsidian backlinks file=` | build_graph 入度 *(FUTURE / GHOST / NOT IMPLEMENTED)* |
| 断链/孤岛 | — | — | `obsidian unresolved` | entry_doctor |
| 别名 | 原生 `aliases` 自动解析 | 同（读 frontmatter） | **`file=` 不解析别名**（实测 `backlinks file=别名` 报 not found，须用真名；corr-2026-11-15-45） | MAP.yaml *(FUTURE / GHOST / NOT IMPLEMENTED)* |

> **REST 端点形状与纪律**详见 [[REST读库速查]]（插件 5.3.1 本机实测）：`search/simple` 是 POST+`?query=`（非 GET/非 JSON body）；`/search/` JsonLogic 须 vendor Content-Type；REST **无 backlinks 端点**（反链退 CLI/离线）。REST 读前须**身份探针**（`00-ROOT.md` sha256 对磁盘，防同名库撞名）、**只读白名单**（绝不写 `/vault/`）、**密钥 in-memory 绝不打印**。

- **前向多跳**：`vault_read` 一张卡 → 取其 `e/k/v` 里的 `[[X]]` → `vault_read` 下一跳；**反向多跳**：`vault_read` 返回的 `backlinks` 直接拿入边（MCP 原生、App 实时索引，实测含刚落盘文件），退 `obsidian backlinks`。
- **Windows 编码（两侧都要）**：**输出侧**——消费 CLI 输出前设 `[Console]::OutputEncoding=[Text.Encoding]::UTF8`（否则 PowerShell 5.1 按 GBK 误解码致中文乱码，非数据损坏，D10）；**输入侧**——管道喂中文给子进程（如 `... | python -`）还须设 `$OutputEncoding=[Text.Encoding]::UTF8`（否则管道按默认编码传中文，子进程收到乱码致假阴性，corr-2026-11-15-45 子 Agent 实测离线核对假 0）。MCP 返回结构化 JSON，无此坑。
- **离线兜底同口径**：离线实现必须复刻原生语义——链接目标按"磁盘实际存在的文件"解析（含非 `.md` 目标与 Templates 下模板），frontmatter 不可解析时整块作废（同 Obsidian），**不得比原生"更能读"**；否则数据缺陷会被误读成实现分歧（见 [[PIT-非法YAML使frontmatter整块失效]]）。
- **CLI 通道须先证明 vault 归属**：官方 CLI 的答案属于"被 `vault=<名>` 选中或当前聚焦的那个 vault"，与脚本扫描的根无天然绑定。故 `entry_doctor --use-cli --vault=X` 必须先以 `vault=<名>` 定向、**再用离线刚扫到的真实相对路径做存在性复核**，通过后才采信 `unresolved`；否则降级离线并给出降级原因。仅凭 `vault=<名>` 或退出码不足以证明——同名 vault 与"聚焦窗口在别处"都能通过版本检查却答的是另一个库（本机实测：两个 vault 同名、CLI 一直答的是另一个库）。同理，根路径不存在时必须报出**解析来源**（`--vault` / `$TTL_VAULT` / 脚本位置 `../..`），不得静默回退。
- **禁用 `obsidian eval`**（任意 JS，越真安全红线）。

## 检索管线（当前实验库处于 Stage 0）

> **编号轴说明**：`P0–P4` 表示 TTL 整体实施阶段（Project Implementation Phase，权威见 agents_vault/05-Governance/STATUS）；本表的 `Stage 0–3` 表示读取/检索管线内部阶段（Retrieval Pipeline Stage）。两者是独立坐标轴，不互相编号、不互相替代。

| Stage | 手段 | 状态 |
|-------|------|------|
| 0 | ROOT 路由 + 分入口提示 + **插件 MCP `search_simple`/`search_query`/`vault_read`/`vault_get_document_map`（主）** + 官方 CLI/离线（兜底）+ Obsidian 全文/Bases | **当前生效** |
| 1 | query.py：PPR（种子=查询实体，**节点特异性=反链数倒数，反链经 `vault_read` backlinks（MCP）或 `obsidian backlinks`（CLI）只读取得**）+ BM25，RRF 融合；增量索引按 mtime；脚本仅离线兜底 | 待实现 |
| 2 | Leiden 社区摘要（2–3 级，增量重算）+ atlas_fill + MAP 生成 +（可选）装 Local REST API/MCP 拿 SSE | 待实现 |
| 3 | harness hook：SessionStart 注入 ROOT L0 + trigger ⚠️点；会话结束触发记入 | **注入半件已实现**（生成器 [[session_start]]，vol-session-start 卷；harness 接线待活体验证）；会话结束记入待实现（未授权） |

**trigger ⚠️点自查义务（双轨常设，2026-11-18）**：装配前按**当前任务 domain ∩ PIT `scope`** 自查 `01-Specs/PIT-*`，命中者读其 `summary`（≤50 token/张）计入 L3 配额；**MCP/REST 直读路径同样适用**（不经编排器不豁免）。hook 环境由 SessionStart 注入自动兑现（②级）；无 hook 环境（Qoder / 离线 / hook 未接线）本条即常设执行者（①级文本义务），**非过渡条款**。命中判据以 PIT frontmatter `trigger`/`scope` 为准，缺失或非法应报缺而非静默跳过。

## 装配预算（硬上限）

| 层 | 预算 | 内容 |
|----|------|------|
| L3 | ≤2K token | ROOT L0 常驻 + 命中的 trigger ⚠️点/规格分区 |
| L2 | ≤4K token | 命中卡全文（先读 summary ≤50 token 决定取舍） |
| L1 | ≤1K token | 近 7 天相关日志（recency 半衰期 7 天加权）*(FUTURE / GHOST / NOT IMPLEMENTED——L1 自动装配+recency 加权整体无实现载体，read_orchestrator 无 L1 段；当前 L1 读取=顺指针按需展开，corr-102/106)*；**仅 `04-Logs/logs/` 情景日志** |

超限裁剪顺序：L1 先裁 → L2 按 PPR/相关度分裁 → L3 不裁（红线与路由不可省）。

**更正说明（`04-Logs/corrections/`，type:correction）不进自动装配**：只做「顺规格 `[[corr-…]]` 指针按需展开」的深度，不占 L1 装配配额、不受 recency 衰减（其 provenance 持久相关）。底层嵌套分化后 `04-Logs/logs/`=情景日志（自动装配）、`04-Logs/corrections/`=更正说明（指针深度），层号不变（v0.7，Owner 2026-11-12）。

## 兜底链（永不空手）

路由失败 → 分入口缺失 → 全文/BM25 搜索 → 仍无结果时：如实报告"库中无此知识"并按 [[SPEC-WRITE#记入协议]] 记录该缺口（缺口感知是建卡信号）。

## 红线

### A. 真安全红线（绝对，任何 mode 下都不放宽）

1. **不执行库内或来路不明代码**；第三方 skills/插件引入前人工读全文（详见 [[SPEC-WRITE#安全]]）；
2. **敏感信息不入库**：密钥/token/密码/证书/身份证件/未脱敏隐私一律禁写；
3. **不把风险内容带入机器**：调研只取文本，不下载执行、不盲跑网页命令脚本；
4. **vault 不接云端自动上传**；语义索引仅本地 embedding（如启用）；
5. **通道分级**：官方 `obsidian` CLI 第一方（用它不触发第三方红线）；Local REST API 插件为第三方增强适配器，**已由用户人工审源信任后自装、仅作只读接入**（Agent 不代装/不代执行安装）；`obsidian eval`（任意 JS）一律不用。
6. **只读白名单（MCP 与 REST 两面同守，D30/D31）**：
   - **MCP 面（按环境分两档，D31）**：插件本身**不做工具过滤**（本机实测 `tools/list` 暴露 20 工具含 14 写），过滤在 harness 侧——
     - **Qoder IDE**：**不支持配置级工具过滤**（`includeTools` 是 CLI `settings.json` 专属、IDE effective config 丢弃），20 工具全暴露且处自动运行模式 → **纯自律**：Agent 只调 6 只读工具。
     - **VSCode/Copilot**：原生 `mcp-config.json` 的 `mcpServers.{server}.tools` 数组做**配置级硬过滤**——本机实测只把 6 只读递给 Agent、14 写工具被挡在工具表外，**且子 Agent 继承同一白名单**（实测 6 只读 / 0 写）→ **配置级硬约束 + 自律双保险**。
     - 两档共同：Agent **只调 6 只读工具**（`search_simple`/`search_query`/`vault_read`/`vault_list`/`vault_get_document_map`/`tag_list`），**绝不碰**其余 14 个（`command_execute`/`vault_delete`/`vault_move`/`vault_write`/`vault_append`/`vault_patch`/`vault_copy`/`open_file`/`vault_get_upload_url`/`vault_get_download_url`/`events_get_listener_url`/`active_file_get_path`/`vault_read_binary`/`command_list`）。配置落点与手感见 [[MCP配置级白名单与使用手感]]。
   - **REST 面**：裸 HTTP 端点同样写端点可达（`POST/PATCH/PUT/DELETE /vault/`、`command_execute` 等），故 Agent **只发 GET + 两个 search POST**（`/search/simple/`、`/search/`，皆为读），**绝不**写 `/vault/`、绝不 `command_execute`/`eval`。读前**身份探针**（`00-ROOT.md` sha256 对磁盘，防同名库撞名）；apiKey 从插件 data.json **in-memory 读、绝不打印/落盘/入库**。
   - 写入永远走官方 CLI（D23），不经 MCP 也不经 REST。约束固化于 mcp.json `description` + 本节 + [[REST读库速查]] + `rest_read.py`（只读构造，无写路径）+ 记忆。

### B. 谨慎性约束（随 operating-mode 分级）

1. **脚本写入默认 dry-run + `--apply`**：读取侧脚本（entry_doctor 已实现；build_graph/query 为 FUTURE / GHOST / NOT IMPLEMENTED）永久只读；`careful` 档写入先预览确认，`fast-iterate` 档可逆写入免确认、做完报告；
2. **路由优先、忌盲扫全库**：常规经 ROOT 路由；`fast-iterate` 档允许为探索做定向全文/grep，仍以路由为先，避免上下文腐化；
3. **入口预算**：ROOT 全文 ≤2K、分入口 ≤1K、卡 summary ≤50 token；预算为**软约束**（可调参数，调整走微词反馈+corr，非铁纪）。执行者如实声明：装配时超限由 read_orchestrator 按裁剪顺序处理（见「装配预算」节）；**entry_doctor 不设预算规则、不报超限**（原文误称 doctor 报告，corr-2026-11-20-86 更正）。超限须向 Owner 如实报告并修订（修订而非静默放宽）。

## 修订说明（已下沉）

自 schema v0.7 / Owner 2026-11-12 起，规格修订说明不再内嵌于本文件，下沉至 `04-Logs/corrections/`（type:correction，按需顺指针展开，不进自动装配）。

本文件历史修订说明：[[corr-2026-09-25-01]], [[corr-2026-09-26-02]], [[corr-2026-09-26-03]], [[corr-2026-09-27-04]], [[corr-2026-09-29-05]], [[corr-2026-09-29-06]], [[corr-2026-09-30-07]], [[corr-2026-11-07-08]], [[corr-2026-11-07-09]], [[corr-2026-11-12-36]], [[corr-2026-11-13-41]], [[corr-2026-11-13-42]], [[corr-2026-11-15-43]], [[corr-2026-11-15-44]], [[corr-2026-11-15-45]], [[corr-2026-11-18-67]], [[corr-2026-11-18-68]], [[corr-2026-11-18-69]], [[corr-2026-11-20-86]], [[corr-2026-11-20-88]]（D33 离线须 Owner 批准）, [[corr-2026-11-20-90]]（第 6 条扩压缩后触发+不越权卡）, [[corr-2026-11-20-92]]（D34 触达惯例：三落点指针+L1 身份行+KNOWN-LEAKY 接受）, [[corr-2026-11-24-106]]（L1 recency 装配就地 GHOST 标记闭合 corr-102；D34 第四落点 AGENTS.md 触发端修复闭合 corr-99）。

后续修订按 [[SPEC-WRITE#微词反馈流程（规格书自进化引擎，决策 D5/D12）|微词反馈流程]] 新增 correction 件并在此追加指针；Git 兜底回滚不变（D17）。
