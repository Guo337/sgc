---
id: "spec-govern-001"
layer: 3
type: spec
spec-kind: govern
trigger: always
project: memory-lib
domain: [memory-governance]
maturity: seed
created: 2026-09-27
last-touched: 2026-11-24
summary: 治理规格书：定义"谁能改库、怎么并入"——开源式 branch→审核=并入→main、申报报告、doctor 闸门、角色与权限
e:
  - { target: "[[SPEC-WRITE]]", rel: depends_on, weight: 0.8 }
  - { target: "[[00-ROOT]]", rel: grounds, weight: 0.5 }
---

# SPEC-GOVERN 治理规格书 v1.0.0

## 职责边界

定义**变更如何被审查与并入 main**——贡献→审核→并入的治理流程、角色与权限、申报报告、doctor 闸门。读取侧见 [[SPEC-READ]]，写入动作细则见 [[SPEC-WRITE]]；本规格管的是"谁能改库、怎么改才作数"。

## 治理模型：开源式 branch → main

借用开源项目管理范式（贡献者提 PR → 维护者审查 → 合并主线）。本地单库以 **Git 分支**实现 fork 语义，无需真 fork/worktree。

### 角色（2 个，Reviewer 与 Maintainer 已合并）

| 角色 | 职责 | 权限 |
|------|------|------|
| **工作 Agent（贡献者）** | 执行任务后在**分支**上产出记忆文件增改 + **申报报告**，commit 到分支、提请审核 | 分支上建卡/追加日志/改规格；**不直接并入 main** |
| **审核=并入 Agent（Reviewer+Maintainer 合一）** | 读申报报告 → 跑 doctor 闸门 → 查冲突/陈旧/孤岛 → 决定并入 main（带原因/触发 log id/diff/回滚点）或退回（退回理由记入审核方当日日志，不回改申报报告） | **merge 到 main 的唯一决策者**；对 90-Meta/schema 的并入把关 |

> **角色分离原则**：产出者 ≠ 批准并入者——打破"自己给自己判卷"（对应评审 F 点）。

### 变更流程

1. 工作 Agent 在分支 `contrib/{topic}-{date}` 上完成记忆文件增改；
2. 写**申报报告**（见下），与文件增改一同 commit；
3. 提请审核=并入 Agent；
4. 审核方跑 **doctor 闸门** + 审阅申报报告 + 查与既有卡冲突；
5. **过 → merge main**（commit 带原因/触发 log id/diff/回滚点）；**不过 → 退回分支，审核方在当日日志记退回理由**（申报报告不可篡改、审核结论不回写报告）；
6. 用户在 main 上可随时 Git 回退；重大裁决 / golden-set 终审归用户。

### 申报报告（取代传统 issue）

由**执行任务的工作 Agent**在提交变更时一并编写（非独立第三方开 issue），声明：

- 改了哪些文件（增/改/删）+ 对应 `[[wikilink]]`；
- 依据来源（log id / 外部来源 / 本机实测）；
- doctor 自检结果；
- **冲突与存疑**（与既有卡的矛盾、不确定处，如实列、不洗白）；
- 建议关系（新卡的边）。

> 模板 `Templates/T-report.md`。审核方据报告复核，报告不实由审核闸门拦截。
>
> **申报报告 = 贡献者的不可篡改声明**（一经提交任何人不得回改，含审核方）。审核结论**不进报告**，记在 **①merge 提交信息**（原因/触发 log id/diff/回滚点）**②审核=并入 Agent 当日日志**（含退回理由）——以此解“报告不可篡改 vs 审核批注”矛盾（第三份评审 #1，走 C 方案）。

### 提交信息规范（commit message）

所有提交——分支上的贡献提交、审核方的 merge 提交、过渡模式的直提 main——标题与备注都须**清晰概括**：

- **标题**：点明主题与范围（例：“治理规格落地：SPEC-GOVERN + schema v0.6”）；
- **备注**：按“新增 / 修订”或按文件 / 决策项**分组分条**，说明改了什么与为什么，忌挤成一整段 run-on；
- **merge 提交**备注额外带：原因 / 触发 log id / diff 摘要 / 回滚点（见变更流程第 5 步）；
- **PowerShell 落地**：多段备注用**多个 `-m` 参数**分段承载（避免单参数内嵌换行的编码坑）；特殊符号（如 ≠ → ↔）易在参数编码中丢失，改用文字或 ASCII。

- **提交节奏（何时提交）**：过渡模式（直提 main）下，每完成一个可验证单元（一个批次 / 一次自洽改动）即自行 `git commit`，不攒批、不等用户逐次授权（用户 2026-09-28 明确“每做一大步记得自己提交”）；提交前必跑 doctor 离线闸门（0 ERROR）。

## 工具脚本规范（90-Meta/scripts）

关键脚本（`agents_vault/90-Meta/scripts/` 下工具，如 `entry_doctor.py`、`reflect.py`）的**注释与 docstring 一律用英文**（用户 2026-09-28：“关键脚本均不可用中文注释”）。以下**保持中文**——属功能字符串/库内容，非注释：

- 脚本的**用户界面输出**（`print` 报告文案、issue 的 `message`/`fix` 修复方向）；
- 脚本**生成的 vault 内容**（如卡片模板正文——vault 本身是中文库）；
- **匹配中文 token 的正则/数据常量**（如 `LOG_SECTIONS`、卡型关键词、self-test fixture 模拟内容）。

存量中文注释逐步改英文；新建关键脚本即遵守。落地实证：`reflect.py` v0.2.0 + `entry_doctor.py` v2.2.0 注释/docstring 已全英化（输出/fixture 数据保持中文）。

## 分支切换与 Obsidian 索引一致性（运行时前置）

branch 模型引入一个“直提 main 时代”不存在的风险：**Obsidian App 开着时 `git checkout` 换分支，磁盘文件已变，但 CLI（`backlinks`/`unresolved`）查的是 App 内存索引，可能滞后于工作树**，导致 doctor 拿旧索引做出错误 merge 判断（第三份评审 #3）。约定：

1. **换分支前先关 Obsidian**，或换后**重载 vault / 重启 App**，待重新索引完成再跑 doctor；
2. **doctor 起跑前的一致性前置（P0-b，规约优先）**：CLI 无法直接查“App 索引对应哪个 commit”，故**不实装为硬闸门**；改为规约（换分支/大改先重载 vault 再跑）+ doctor `--use-cli` 报告与 `--json` 均标注“对账基于 App 内存索引，可能滞后工作树”的软提示（v2.2.0 起）。**用户裁决（2026-09-28）**：时机成熟后仍不排除加一点硬代码检测（如 git HEAD 时间 vs .md mtime 启发式），但硬代码**不一定用作闸门**（可仅告警）；
3. **离线兜底**：索引不一致且无法重载时，doctor 退现有 Python/grep 离线通道（D23/D25 已定义）直接扫工作树，报告标注“离线、未经 App 索引”——一致性优先，非重造链接引擎。

> **迭代期特批（2026-09-28）**：doctor 大半职能上下文无关（纯标准库、只读扫工作树），用户特批 Agent **自跑 doctor 闸门**、无需逐次请示；与 §提交信息规范“提交节奏”（提交前必跑离线闸门）一致。

## doctor 闸门（merge 前必过）

`entry_doctor`（真值源=脚本卷 `agents_vault/90-Meta/script-volumes/entry_doctor.md`，只读、纯标准库；**P0 doctor gate 已落地 [[entry_doctor#VERSION|v2.17.0]]**）作为并入 main 的 CI 闸门。**信任前置（v2.7.0，独立评估 S1 闭合）**：信任一次扫描结果前，须先对同一抽取件跑 `--self-test` 且 rc=0——sha256 是自证锚（能改库者能改 doctor 并重算 sha），self-test 是真破坏性探针（阉割版必 MISS），强制前置即 POST 式自检；doctor 卷任何代码改动必附 correction。外部信任根=git 历史 + Owner 裁判，不做外置签名。

**运行方式（“做Doctor”）**：.py 无常驻文件（gitignore，用时抽取）——先从卷二进制抽取 .py（开围栏 ````python 定位、闭围栏 rindex、切片 data[s:e]、比 frontmatter sha256），再执行 `python <抽取件> --vault <库根>`（抽取件放 scripts/ 下时省略 `--vault` 可自动定位库根 = `__file__` 的 `../..`）。**抽取件运行完毕后删除（含 `__pycache__`）；`scripts/` 常驻 .py 仅限冻结实验仪器（见 [[corr-2026-11-16-56]]），runtime 抽取件不得常驻**（「用完即删」动作指令，corr-56；此前仅 gitignore 注释承载，规范文本缺位致跳步实证）。**默认离线扫工作树**——一致性优先，且 App 未运行时兜底；**App 在跑时加 `--use-cli`** 叠加官方第一方 `obsidian` CLI（D23，**可完全依赖的主通道**）做 `version`/`unresolved` **自动对账**（分类：一致/离线误报/CLI 漏判/非 wikilink 目标，真分歧升 WARN；**v2.2.0 起过滤 CLI 空结果哨兵串** `No unresolved links found.`，避免误判断链目标升假 WARN），CLI 不可用则优雅降级离线（本机实测：批次1 清净后库内无 unresolved，`obsidian unresolved` 返回空结果哨兵→v2.2.0 正确过滤、自动对账四项皆“无”、闸门 0/0/0；App 关→`TimeoutExpired` 降级，两态均 PASS）。`--self-test` 跑内置破坏性 fixture 自检（全部注册规则 + BROKEN_SET + ID_NAMESPACE + VOL_RULES + VOL_DEPS + GOV_RULES + OV_RULES + HUB_RULES + DOC_IMPL_DRIFT + PARSER_V27 + CLI_RECONCILE + CLI_VAULT_PIN + CLI_UTF8_DECODE，应全 OK）；`--json` 输出稳定机读 schema。退出码 `0`=无 ERROR / `1`=有 ERROR / `2`=工具自身失败。见 [[corr-2026-11-16-49]]。

检查项（括号内为 rule id）：

1. YAML/frontmatter 可解析（`FM_MISSING`）：严格扁平子集（schema `frontmatter_subset`）——重复键、块标量、嵌套映射、锚点/别名、tab 等子集外形态整块判不可解析（与 Obsidian 丢弃整块同判，fail-loud 不静默近似）；
2. common 必填字段齐全（`FIELD_MISSING`）：空白串/null/空列表均判缺；
3. `id` 唯一（`ID_DUP`）+ 格式合规（日志/报告按日编号例外）；
4. **type↔layer 合法组合**（`TYPE_LAYER`/`LAYER_BAD`）：L1→log/report/correction；L2→concept/claim/procedure/observation/entity/summary；L3→spec/pitfall/feedback；
5. `type:spec` 带合法 `spec-kind`（`SPEC_KIND`）；
6. `summary` ≤ 预算（`SUMMARY_LONG`，WARN）；
7. **断链 broken-link**（`BROKEN_LINK`，ERROR）：默认离线扫工作树，**wikilink + markdown .md 双形态**；**剥离代码跨度/围栏块**用 CommonMark 子集行扫描（开围栏须行首、info string 禁反引号、闭围栏长度≥开围栏、未闭合延伸到 EOF），对齐 Obsidian“代码内不成链接”语义（D21）——工作树一致性优先；库外目标（../ 越界）按磁盘存在性判；`--use-cli` 叠加 `obsidian unresolved` **自动对账**（按可比性分类，详见下“CLI 自动对账”节）（App 未运行/索引不一致时优雅降级）。离线不重造完整链接引擎（B 决策边界内）；
8. **L2 边约束**（`EDGE_ORPHAN`，ERROR）：非 exempt 的 L2 卡至少一条 `k/v/e`；`edge-policy: exempt`（Atlas summary）豁免 frontmatter 边，改判“auto 区块存在且非空（`ATLAS_EMPTY`，ERROR）+ 其列出链接可解析 + ROOT 可达”；
9. **不可达 unreachable**（`UNREACHABLE`，WARN）：从 00-ROOT 出发 BFS（沿 wikilink / md 链接 / 目录链接），不可达者告警；log/report/correction/pitfall/**overview** 豁免（记录/触发式发现/指针深度/纯人读地图，非 ROOT 导航目标；overview 故意不挂路由，不豁免则恒报 WARN 自绊 0E/0W/0I，corr-2026-11-20-89）；
10. **stale 陈旧**（`STALE`，INFO，不阻断）：仅 L2；信号 = 被 `rel:supersedes` 指向 或 `last-touched` 超 180 天；
11. **重复**：`id` 重复（ERROR）/ 标题·别名冲突（`DUP_TITLE`/`DUP_ALIAS`，WARN）；
12. 日志正文含四固定段（`LOG_SECTION`，仅 type:log）；
13. **更正说明合规**（`CORR_SECTION`/`CORR_TARGET`，ERROR，v0.7）：type:correction 须含三段（改了什么/为什么改/依据）+ frontmatter `target` 必填；落点 `04-Logs/corrections/`；
14. **规格禁内嵌 changelog**（`SPEC_CHANGELOG`，ERROR，v0.7）：type:spec 正文不得含 `## changelog` 实体段——修订说明应下沉至 correction，规格只留当前态 + 「修订说明（已下沉）」指针节。
15. **id 命名空间合规**（`ID_NAMESPACE`，ERROR，v2.3.0）：声明了 `id-namespace` 的副库，新建条目 id 须以 `{kind}-{slug}-` 起首；复制上游的条目保留原 id 但须标 `provenance: copied`（豁免）；主库（无声明）豁免。详见 schema `id_namespace`。
16. **脚本卷替代判据**（`VOL_FM`/`VOL_FENCE`/`VOL_SHA256`，ERROR，v2.4.0）：type:script-volume 豁免知识必填/layer/type↔layer（G 原理），改判——必填技术字段齐 + id 形态 `vol-{program}` + id↔文件名↔source-of 同源（VOL_FM）；恰一个四反引号 python 代码块、围栏顶格完整（VOL_FENCE）；代码块实算 sha256 == frontmatter 声明（VOL_SHA256）。卷代码块是不透明程序文本，其 wikilink 形字符串不进链接图。详见 schema `script_volume`（[[corr-2026-11-16-48]]）。
17. **实现漂移对账 G20**（`DOC_IMPL_DRIFT`，v2.4.0 起；v2.7.0 收缩为单锚点）：链接携带意图——当前态引用写成 `[[卷#VERSION|别名]]` 活链接，对账器从卷代码块 VERSION 常量取真值与别名宣称值比对，漂移 **ERROR**；纯文本提及=历史归属，天然豁免。锚点名大小写不敏感；无别名/别名无版本号 → WARN；别名含多个版本号 → WARN（无法判定宣称）；预发布后缀（-beta 等）参与比对。**RULES/SELFTEST 锚点已删**（v2.7.0 TCB 收缩：装饰性计数+可伪造真值通道，独立评估 A+B 合流裁定；文档不再宣称具体规则数/自检项数）。见 [[corr-2026-11-16-54]] 及本轮 corr。
18. **卷依赖图校验**（`VOL_DEPS`，ERROR，v2.5.0）：`depends-on` 可选字段，若存在须为 vol id 列表（依赖图主键，Owner 2026-11-16 裁②A）+ 目标卷存在 + 无循环（DFS 检测）。抽取器 [[vol_extract]] 按此递归拓扑抽取；联邦复制时依赖卷须一并复制。详见 schema `script_volume.optional`（[[corr-2026-11-16-51]]）。
19. **卷名唯一 + 封闭世界**（`VOL_NAME_DUP`/`VOL_KNOWLEDGE_FIELD`，ERROR，v2.7.0）：卷 basename 全局唯一（影子卷不得覆盖 `[[名#VERSION]]` 真值源）；卷禁带知识字段（layer/k/v/e/edge-policy/project/domain/maturity/summary/aliases 等）——豁免是双向的：卷豁免知识判据，也不得穿戴知识字段。规则适用性由引擎按类别集中分发（knowledge/volume/all），不再靠各规则体内守卫。
20. **copied 来源佐证**（`ID_NAMESPACE` WARN 分支，v2.7.0）：provenance:copied 豁免前缀校验须带 `source` 指针（上游条目 wikilink 或 id）；缺佐证升 WARN 不拦死（真实联邦复制不阻断，但缺失可见）。
21. **契约本体对账**（`CONTRACT_DRIFT`，ERROR，v2.7.0）：schema.yaml 是字段表/枚举的宣称权威，doctor 常量是实现——扫描时对账 common 必填表、script_volume.required、governance.required（v2.8.0）、type↔layer 映射，漂移 ERROR。闭合「schema↔doctor 双权威无对账路径」（独立评估 A 最高风险）。schema 不可读 → WARN。
22. **治理族替代判据**（`GOV_FM`/`GOV_STATUS`/`GOV_KNOWLEDGE_FIELD`/`GOV_SINGLETON`/`GOV_ENTRY`，全 ERROR；GOV_ENTRY 批4 升格 v2.11.0，[[corr-2026-11-16-61]]）：type ∈ {constraint, reference, status} 的治理族节点（Agent-Governance 内迁，[[corr-2026-11-16-57]]）豁免知识必填/layer/type↔layer（G 原理，卷判例同款），改判——必填+条件必填齐（按 type 分派）+ id 形态 gov-{yyyy}-{slug}（status 单例 gov-status）+ priority/scope 枚举（GOV_FM）；status 枚举合法 + SUPERSEDED 须 supersedes 指针且目标存在（GOV_STATUS）；禁带知识字段（封闭世界双向豁免，aliases 合法区别于卷）（GOV_KNOWLEDGE_FIELD）；type:status 全库恰一个（GOV_SINGLETON，主库批3 起 presence 恰 1；**声明 id-namespace 的副库豁免存在性**——治理族 federation:never，副库自建 STATUS 自定节奏，v2.16.0，[[corr-2026-11-24-105]]）；经 05-Governance/00-INDEX 导航锚可达（GOV_ENTRY，批4 入口内迁后 ERROR）。STATUS 当前态宣称机械对账（STATUS_CLAIM 6 项，ERROR；其中主库历史锚 log 检查对副库豁免，v2.16.0）+ 正文 ≤200 行（STATUS_SIZE，WARN），v2.10.0，[[corr-2026-11-16-60]]。类别引擎四分（knowledge/volume/governance/overview），知识规则不穿透治理族/概览；ID_NAMESPACE 豁免 gov-*/vol-*/ov-* 独立前缀（联邦边界：副库见治理族/概览判「非本库本体」不报错）。GOV_NAME 暂缓（DP-6 零改名：编号是结构化标识符，无可计算命名判据则不立规则）。详见 schema `governance` 节。
23. **枢纽抽象轴**（`HUB_GROUNDS`/`HUB_DECOMPRESSION`，WARN；`SECONDARY_TYPE`，ERROR/WARN，v2.9.0）：hub:true 的 L2 卡须有≥2 条强语义入边（implements/grounds/specializes，derives_from 不计=来源非实例）（HUB_GROUNDS）；须声明 abstracts 解压路径边且每条 abstracts 须有对应 implements 反向边（互逆对账，DP-3 乙）（HUB_DECOMPRESSION）。secondary-type 双 type 过渡（DP-4）：合法知识枚举 + ≠type（ERROR）+ 正文须含分化计划指针（WARN）。hub 标记是结构事实（入度≥2 即标，DP-5 方案A），深浅归升迁轴不混判；资格语义归 Owner，机械只判声明与证据一致（WARN 非 ERROR）。abstracts rel 新增（implements 的逆，schema rel_enum）。详见 schema `L2_edges.hub`/`secondary-type`（[[corr-2026-11-16-58]]）。
24. **全景概览替代判据**（`OV_FM`/`OV_KNOWLEDGE_FIELD`，ERROR，v2.13.0，[[corr-2026-11-20-89]]）：type:overview 的纯人读项目地图（README，落点 03-Atlas/README.md，overview 内迁）豁免知识必填/layer/type↔layer（G 原理，卷/治理族判例同款），改判——必填 id/type/created/last-touched/summary 齐 + id 形态 ov-{yyyy}-{slug}（OV_FM）；禁带知识字段（封闭世界双向豁免，aliases 合法）（OV_KNOWLEDGE_FIELD）。**第四类别**（knowledge/volume/governance/overview），类别引擎中央分派；UNREACHABLE 显式豁免（纯人读故意不挂 ROOT 路由）；断链沿 BROKEN_LINK(ERROR, all) 判不另立 OV_LINK（G 原理要求替代判据不要求重复判据）；不进检索装配（read_orchestrator 类别过滤执行，非仅声明，E 原理）、不结晶不升迁不进副库。详见 schema `overview` 节。
25. **pitfall 触发契约**（`TRIGGER_FM`，ERROR，v2.14.0，[[corr-2026-11-20-95]]）：type:pitfall 的 L3 卡须有 trigger（枚举 {always, on-domain, on-file-pattern, on-project}，真值源=schema L3.trigger）+ scope（首版只判非空，不校验与 trigger 语义匹配，留后续刀）；缺 trigger/scope 或 trigger 非法枚举→ERROR。与 session_start 生成器运行时 fail-loud 孪生（CORRECT-1：生成器自带同款守卫不等 doctor，F1 是共时性结构看守补位）。存量 13 张 PIT 全合规上线即绿。
26. **SH 投影对账**（`ATLAS_GROUP`，WARN，v2.15.0，[[corr-2026-11-21-96]]）：C-1 形态 SH 域入口（文件名 SH-{domain}+auto 区块+首注释 C-1 派生规则，域从注释解析/文件名回退）三件对账——①枢纽组成员==hub.abstracts∩C-1（多归属每组都列）；②未聚簇==C-1−枢纽组实列并集−hub 自身（2A 豁免，corr-93）；③C-1 收录集==清单全集（corr-84 抓漏机械化）。失配 WARN 非 ERROR（SH 是派生视图，失配=该重跑生成器非知识损坏），fix=跑 atlas_fill 核 diff 后 --apply。与 atlas_fill 输出同定稿形状；行序不对账（集合对账，排序归生成器）；非 C-1 形态 SH（SH-index/角色件）不适用。

**每条 issue 带 `rule` + `fix`（修复方向）**，满足 P0 验收“报告指出文件 + 规则 + 修复方向”。输出：稳定 `--json`（tool/version/vault/scanned/mode/counts/issues/cli_reconcile）+ 人类可读报告；退出码 `0` 无 ERROR / `1` 有 ERROR / `2` 工具自身失败。

**`--self-test`**：内置破坏性 fixture（内存、不写盘），验证全部注册规则可被触发 + BROKEN_SET（断链目标入集）+ ID_NAMESPACE（命名空间正反例）+ VOL_RULES（脚本卷契约正反例）+ VOL_DEPS（依赖图正反例：合法链豁免，缺失/形态/循环各触发）+ GOV_RULES（治理族契约正反例：干净节点零告警豁免知识字段，GOV_FM/STATUS/ENTRY/KNOWLEDGE_FIELD/SINGLETON 各触发，知识规则不泄漏，副库 gov-* 豁免 ID_NAMESPACE）+ OV_RULES（概览契约正反例：干净 overview 零告警豁免知识字段+UNREACHABLE 豁免，OV_FM/KNOWLEDGE_FIELD 各触发，知识规则不泄漏，副库 ov-* 豁免 ID_NAMESPACE，v2.13.0）+ HUB_RULES（枢纽契约正反例：干净 hub 零告警，入度不足/无 abstracts/缺逆边各触发 WARN，secondary-type 非法/相同 ERROR/缺计划 WARN）+ TRIGGER_FM（pitfall 触发契约正反例：缺 trigger/缺 scope/空 scope/非法枚举各 ERROR，合规卡零告警，v2.14.0）+ ATLAS_GROUP（SH 投影对账正反例：组缺卡/未聚簇混入枢纽卡/未聚簇缺卡/C-1 漏收/多收各 WARN，干净投影零告警，域名非硬编码，v2.15.0）+ DOC_IMPL_DRIFT（VERSION 锚点正反例：漂移 ERROR 含小写/预发布/路径形态，多版本/无别名 WARN，正确链接/历史纯文本豁免）+ PARSER_V27（解析层正反例：幻影围栏不吞活链/EOF 围栏保持代码/空白 null 必填判缺/重复键与子集外整块作废/md 断链检出）+ CLI_RECONCILE（对账纯函数分类/分歧检出，无需 App）+ CLI_VAULT_PIN（vault 定向与错误回复判别）+ CLI_UTF8_DECODE（CLI 中文 UTF-8 输出正确解码，v2.2.1）（借 SwarmVault smoke / Loreweave 回归的测试方法）；当前全 OK。

**CLI 自动对账（v2.2.0，`--use-cli`）**：App 在跑时 `reconcile_cli()` 把 `obsidian unresolved` 原始输出与离线断链集**按可比性分类对账**（离线只扫 `[[wikilink]]`，CLI 还含目录/markdown 链接/embed，故不裸集合相减）：① 一致断链；② 离线有 CLI 无（疑离线误报或 App 索引滞后）；③ CLI 有 离线无（wikilink 类，疑离线漏判）；④ 非 wikilink 目标（目录/markdown 链接，离线不覆盖，仅提示）。②③ 为真分歧升 `CLI_RECONCILE`（WARN，不阻断 merge）；④ 不计入闸门。**v2.2.0 修复（坑B）**：`obsidian unresolved` 无断链时输出字面量哨兵串 `No unresolved links found.`，此前被误当一个未解析 wikilink 目标→升假 WARN；现由 `CLI_EMPTY_RE` 过滤（`--self-test` 加空哨兵用例守护）。结果进人类报告“自动对账”段 + `--json` 的 `cli_reconcile` 块（含 `index_freshness_note` 软提示，见 P0-b）。本机实测：批次1 已把 `03-Atlas/`、`90-Meta/MAP.yaml` 两条不解析链接改 inline code，库内无 unresolved，`--use-cli` 自动对账四项皆“无”、闸门 0/0/0。

> **stale 与 L1 日志**：日志是永久审计记录，只按 recency 衰减检索权重 *(FUTURE / GHOST / NOT IMPLEMENTED——recency 衰减检索无实现载体，corr-102/106)*，**不判 stale**（stale 仅 L2）。
>
> **待补（P0 余项）**：① HEAD/索引一致性（P0-b）已按用户裁决 A 落地为**规约 + doctor 软提示**（v2.2.0 `--use-cli` 标注索引可能滞后），**非硬闸门**；硬代码检测（git HEAD vs mtime 启发式）留未来选项，且不一定作闸门（见 §分支切换）。② “故意破坏分支被拦”的端到端演练待角色实例化后跑（P0-c；(b) 子 Agent 审核为**倾向方案**，评审面已在 G1/G2 实操（起草→独立审核→Owner 签），但角色实例化**未正式选定**——权威状态以 §待定决策 #1 为准，2026-11-20 澄清；原"批次5"系旧计划语汇，现行波次见实施计划 r2）。（原 P0-a “--use-cli 离线↔CLI 自动对账”已于 v2.1.0 落地、v2.2.0 修哨兵串，见上“CLI 自动对账”节。）

## 权限矩阵修订（对齐 A 决策）

本规格修订 [[SPEC-WRITE#写权限矩阵]] 一处：**90-Meta/schema 不再对 Agent 绝对禁止**——Agent 可在**分支**上改 schema，经审核=并入 Agent 过 doctor 闸门后并入 main，**Git 全程兜底可回退**（A 决策：直接写入修改、靠 Git 回退）。`MAP.yaml` 仍禁 Agent 改（脚本生成物）。

## 与外部语法的关系（B/C 决策）

分层原则：**Obsidian 管链接与检索（原生 CLI），外部语法管语义与治理**。

- 链接图 / 全文 / 反链 / 断链 → 走 Obsidian 原生，**不重造**（B：放弃不依赖 Obsidian 的平行工具链）；
- 语义 / 治理约束（profile、`rel`、`edge-policy`、type↔layer）→ 用**扁平键值对**承载（Bases 可查），由 doctor / 审核 Agent 校验（C）；
- 记忆内容依赖"外部语法"（Obsidian 不原生理解、由外部工具校验）**可接受且有明显效益**，妥善处理到几无影响。

## 待定决策（需用户拍板）

1. **角色实例化**（当前"单用户 + 单 Agent"测试环境如何跑出独立审核方）：(a) 我一人分饰两角（弱独立）/ (b) 子 Agent 审核（中，**推荐**）/ (c) 用户当审核方（强）/ (d) 仅作目标架构、暂仍直提 main。**选定前本库处过渡模式：直提 main + Git 兜底**（即 (d) 临时态）。2026-11-20 补充：四分身份分类学（[[Agent身份分类学]]）落地后，「审核者↔合一角色映射/merge 权归属」成为本决策的具体形态——Owner 裁 C 案等真实触发（审核者系备用角色短期不开发；张力入分类学卡存疑节，role-ke-auditor 仅评审面）。
2. **golden-set 终审**：检索基线的标准答案由审核 Agent 起草、**用户终审**（F 的可靠性底线，不可自证）。
3. **新字段** `status/confidence/source`：日后逐步规划（D），现不加；`status: pending-review` 待自动结晶（reflect）上线时引入。

## 修订说明（已下沉）

自 schema v0.7 / Owner 2026-11-12 起，规格修订说明不再内嵌于本文件，下沉至 `04-Logs/corrections/`（type:correction，按需顺指针展开，不进自动装配）。

本文件历史修订说明：[[corr-2026-09-27-20]], [[corr-2026-09-27-21]], [[corr-2026-09-27-22]], [[corr-2026-09-27-23]], [[corr-2026-09-27-24]], [[corr-2026-09-27-25]], [[corr-2026-09-28-26]], [[corr-2026-09-28-27]], [[corr-2026-09-28-28]], [[corr-2026-09-30-29]], [[corr-2026-11-12-36]], [[corr-2026-11-13-40]], [[corr-2026-11-16-49]], [[corr-2026-11-16-51]], [[corr-2026-11-16-54]], [[corr-2026-11-24-105]], [[corr-2026-11-24-106]]（recency 衰减检索就地 GHOST 标记）。

后续修订按 [[SPEC-WRITE#微词反馈流程（规格书自进化引擎，决策 D5/D12）|微词反馈流程]] 新增 correction 件并在此追加指针；Git 兜底回滚不变（D17）。
