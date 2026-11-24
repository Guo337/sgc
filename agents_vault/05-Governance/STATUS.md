---
id: gov-status
type: status
status: ACTIVE
scope: workspace-agent
created: 2026-09-29
last-touched: 2026-11-25
aliases: [项目状态, TTL STATUS]
---

# TTL Memory Library — STATUS（gov-status，当前态指针）

> 历史链：本文件只留当前事实；每次变更先落 L1 再改此处（BCP 协议）。
> 3C 期历史 → Audit/Current/TTL-Core-Hypothesis-Spec-NON-AUTHORITY/ROUND-3C-V2-EXECUTION-RECORD.md；
> 裁决链 → 04-Logs/corrections/ + logs/ 指针；孤本 hash 映射 → log-2026-11-16-06。
> Date: 2026-11-24（分区清单签认 corr-113+修订权契约裁决 corr-112+06-Project 工程波 corr-111+v3 裁决 corr-110+Beta r4/r4-b corr-107/109+README corr-108+修复批 corr-106）

## 1. 当前态

- **Maturity: alpha**（Owner 2026-11-14 宣告，log-2026-11-14-01；定义=确定性收束成功，不担保每条结论正确）。
- **Stage**: 语义线 Round 3C 确定性收束已完成（Run C 12/12 PASS，五门全过，AUDITABLE）；治理内迁**收官**（批1~4 全交付，10/10 内迁，旧目录已退役删除——丙案，corr-2026-11-16-62）；**推广波1 验收完成**（2026-11-18，P-A~P-F 全批，corr-2026-11-18-63 / log-2026-11-18-01）；**导航重分配波1 落地**（2026-11-18，刀A L0 指针化 652→294 tok + 刀D trigger 自查义务双轨常设，corr-67/68 / log-2026-11-18-02）；**治理 G1 户口落地波收官**（2026-11-20，Owner 波级签收：八刀全完，联邦 28 逐名=24 旧+4 新，incubator 建区人口 1，四卡剥离升迁，SH link-only 634 tok，散文退役，冻结 8/8 零漂移，corr-71~78 / log-2026-11-20-01/02/03；执行依据=Audit/Current/TTL-GOVERNANCE-IMPLEMENTATION-PLAN-FINAL-2026-11-20.md r2）；**治理 G2 SH 入口立体化波收官**（2026-11-20，Owner 波级签收：四分身份导航落地，SH-index 身份根+4 角色件+ROOT 角色轴，盲测 PASS 三跳达义务，预算全达标 ROOT L0=357≤500，冻结 8/8 零漂移，联邦 28，corr-79~81 / log-2026-11-20-04；G2-6 四角色件义务包补全 corr-82，G2 全刀完成）。
- **Blocker**: None（P0 COMPLETE；P1 first slice DONE；scope freeze 内无阻塞）。
- **Next task**: 〔接续/新立项/压缩后首个动作：先走 [[SH-index]] 认身份→角色件→域 SH（SPEC-READ §入口 6，D34；触发端第四落点=仓库根 AGENTS.md，压缩后仍在，corr-106〕**Beta 工程波**（r4+r4-b+v3 裁决已回，判据 FROZEN，corr-107/109/110）：①06-Project 托管区**已落地**（corr-111：schema v0.7.6+doctor v2.17.0+编排器 v0.4.0+模板入口+权威条款+首件试跑 0E）→②修订权契约**收官**（a–g 裁决 corr-112+首版分区清单 41 单元 Owner 签认 corr-113）→③**re-sync 补建**（diff 式升级协议+上行回流+FED_* 三判据+契约条款入权威层+schema federation-class 落地，端到端样本须含本地偏离，入 B3 门槛；首工程决策=混合级节的字段粒度+上游缓存完整性锚）**〔复杂度质问已处置（corr-115）：载体裁甲=维持已裁 schema 外置清单+框架内简化；清单 v2 重盘已签（corr-117）；**波③已落地**（corr-119：schema v0.7.7 federation_class 49 单元+doctor v2.18.0 FED_* 三判据+fed_sync 卷三方合并+SPEC-GOVERN 契约权威层+端到端 6 阶段 PASS 含本地偏离入 B3 证据）〕**④CI **已落地**（corr-120：doctor-gate.yml 三平台矩阵首跑 run 45453977853 全绿，真 Linux/macOS self-test 首证，B4「CI 矩阵绿」首次机械满足；豁免边界遵守 §5）⑤**已落地**（corr-121：09-BETA-GATE 冻结判据公开件+README 承诺兑现+门面口径刷新 11 卷/re-sync 已落地/CI 已上线；快照 a36f0eb 提交成，push 待网络恢复）。**Beta 五波工程全部落地**；**B3 第二上提已达标**（corr-122：PR Guo337/sgc#2 六步验收并入 c-2026-00037，Owner 裁独立卡+互引）；剩余项=等真实触发（B2 副库使用样本/B3 真实跨库 re-sync 走通）+两条结晶待批。**fed_sync v0.1.1**（corr-123：修零宽插入静默丢弃真 bug——区域行走跳过 insert opcode 致本地偏离丢失，契约级缺陷；+内建 self-test 9 例入 CI 信任前置）。**网络备用通道**：GitHub 直连失败可走 Tailscale VM 推送（Owner 告知，启用前先探活）。**快照已先推 d78adda**（corr-116：rest_read v0.2.1 CRLF 修复外部正等+波①②稳定，不必等波③④⑤；冻结判据发布仍随⑤）。**待外部**：ansys 重测**已闭环**（97 文件 0E，corr-105 豁免验证生效；另报探针 CRLF 真 bug 已修 rest_read v0.2.1，corr-114）；普适卡候选已交 PR Guo337/sgc#2（MERGEABLE，待六步验收）；Linux 评审方真机 self-test 重验（corr-106）。**待办②**：聚焦任务书脱敏+冻结实验件（含脱敏 kit 设计，裁⑥）。前序裁决链与历史叙述不入本文件（BCP）：corr-129/log-2026-11-25-16（issue#5 B 方案：红线节 corr 指针降纯文本，三规则互斥根治）← corr-128/log-2026-11-25-15（README 上提利害+私货边界+引入成本）← corr-127/log-2026-11-25-14（issue#4 处置：同步归维护者+四层 SH 提示，vol_extract v0.1.1/fed_sync v0.1.2）← corr-126/log-2026-11-25-13（doctor v2.18.1 上游快照硬跳过，issue Guo337/sgc#3）← corr-125/log-2026-11-25-12（反馈回路显性化：欢迎 Agent issue+副库回馈节）← corr-124/log-2026-11-25-11（README 语体修正）← corr-123/log-2026-11-25-10（fed_sync 插入丢弃 bug 修复+自检入 CI+README 进度宣称）← corr-122/log-2026-11-25-09（PR#2 六步验收并入，B3 上提达标）← corr-121/log-2026-11-25-08（波⑤判据公开+门面刷新+快照 a36f0eb）← corr-120/log-2026-11-25-07（波④ CI 三平台首跑全绿）← corr-119/log-2026-11-25-06（波③ re-sync 工程落地+端到端样本）← corr-118/log-2026-11-25-05（快照残留误推反思+显式 add 纪律）← corr-117/log-2026-11-25-04（清单 v2 重盘签认，波③前置完成）← corr-116/log-2026-11-25-03（快照 push d78adda+同步工具卷封误伤修复）← corr-115/log-2026-11-25-02（复杂度质问三报告合流+载体裁甲+清单真 bug）← corr-114/log-2026-11-25-01（探针 CRLF 修复+复杂度质问处置）← corr-113/log-2026-11-24-09（分区清单签认）← corr-112/log-2026-11-24-08（修订权契约 a–g 裁决）← corr-111/log-2026-11-24-07（06-Project 工程波落地）← corr-110/log-2026-11-24-06（06-Project v3 裁决）← corr-109/log-2026-11-24-05（B3 扩展修订权契约）← corr-108/log-2026-11-24-04（README 去主库中心化）← corr-107/log-2026-11-24-03（Beta r4 裁决落地）← corr-106/log-2026-11-24-02（外部评审修复批）← corr-105/log-2026-11-24-01（首个外部冷启动处置）← corr-104/log-2026-11-23-02（公开+README 重写）← corr-103/log-2026-11-23-01（理念审计处置）← corr-101/102/log-2026-11-22-04（审计轮二）← corr-99/log-2026-11-22-02（公开准备）← corr-97/log-2026-11-22-01（对外名）← corr-95/96（导航波2/3）← corr-92（D34）← corr-89/91（overview 内迁）← corr-83~86（G1/G2 遗留处置）；各轮细节顺指针展开。
- **Doctor**: v2.18.1（代码真值，entry_doctor 卷 VERSION 常量）；上游快照硬跳过（90-Meta/upstream 非库内容，外部 issue Guo337/sgc#3，corr-126）；联邦修订权 FED_* 三判据（FED_DECL 四态/FED_REDLINE/FED_VERSION 键失锚 fail-loud，仅 copied 规格触发，corr-119）；项目族 PROJ_* 五判据+REPORT_FM（06-Project 托管区，corr-110）；编排器 v0.4.0 过滤扩 overview+治理+项目（治理族既有缺口闭合）；规则计数真值=卷 RULES 注册表（文档不物化具体数，v2.7.0 TCB 收缩裁定；原「25 规则+分组」枚举系物化残留已清，corr-103）；overview 第四类别（OV_*+UNREACHABLE 豁免+编排器过滤，corr-89）；pitfall TRIGGER_FM 触发契约（corr-95）；SH 投影 ATLAS_GROUP 对账（corr-96）；副库 id-namespace 豁免 GOV_SINGLETON 存在性+STATUS_CLAIM 主库历史锚（首个外部冷启动 issue Guo337/sgc#1，corr-105）；自检 CLI_VAULT_PIN fixture 平台门控（外部 Linux 评审，corr-106）。
- **Schema**: v0.7.7（federation_class 三级管辖表 48 单元+upstream_cache 双锚+deviation 四态，corr-119；project 节+REPORT_FM+顶层约束修订，06-Project 已落地 corr-110）；SPEC-READ v0.7.8（D33-b Reader 权限；D34 触达惯例 corr-92+第四落点 AGENTS.md corr-106；D33 离线须 Owner 批准 corr-88；recency GHOST 标记 corr-106）/ SPEC-WRITE v0.9.5（D34 SH 身份行 corr-92；recency GHOST 标记 corr-106）/ SPEC-GOVERN v1.0.0（Owner 2026-11-22 终审摘草案标签升 1.0.0，W4 处置；第 22 条副库豁免 corr-105；recency GHOST 标记 corr-106）。
- **Governance evidence**: dogfooding 23 events / 9 positive / 14 negative / 11 Owner-asked（权威原本=TTL-GOVERNANCE-DOGFOODING-LOG.md §3）。
- **Corrections**: 129 件；Logs: 99 篇（磁盘实测 2026-11-25；计数随库增长，权威真值以磁盘为准）。
- **Frozen instruments**: 8 reducer 在 Audit/History/TTL-Round-3-Superseded/instruments/（甲-2 迁移，corr-56）；scripts/ 0 常驻 .py（卷抽取用完即删）。
- **Baseline**: one-hop oracle #3 冻结于 95b28da（01-BASELINE.md 身份块）。
- 依据：V2-EXECUTION-RECORD §11-§12 / MANIFEST gates_v2 / log-2026-11-14-01 / corr-56 / 01-BASELINE。

## 2. 架构摘要（治理视角）

三层模型 L1（logs/corrections）→ L2（Cards）→ L3（Specs/ROOT）。权威序：Formal Specs > ROOT > STATUS > Cards > Logs。入口：ROOT L0 → sub-entry → atomic card（渐进披露）。Spec 模型：SPEC-READ（读）/ SPEC-WRITE（写）/ SPEC-GOVERN（治理/runtime 边界）。详见 00-ROOT / 01-Specs/（权威原本）。

## 3. 访问模型

见 SPEC-READ v0.7.8（权威原本，含环境分流/REST/两级白名单/降级链；D33 退离线须 Owner 批准；D33-b Reader 权限批准只读离线降级）。不复制快照。

## 4. 实现现实

### CURRENT main path
- Read: MCP read tools → official CLI → offline fallback（SPEC-READ 通道序）。
- Write: append_log（卷）--channel cli → offline fallback。
- Validation: entry_doctor（卷）v2.18.1 / merge gate。
- Read orchestration: read_orchestrator（卷）one-hop + explainable hit path。
- Session injection generator: session_start（卷）ROOT L0 + 全量 PIT ⚠️点，只读 stdout（刀H 生成器半件；harness 接线待活体验证）。
- Shared locator: ttl_locator（卷）；precedence --vault → TTL_VAULT → config → discovery → legacy fallback。
- Formal Specs present in vault。

### EXPERIMENTAL / transitional
- reflect（卷）：非 minimal，含结晶/分化/反思流程。
- append_log --channel rest：非主路径。

### HISTORY / RESEARCH
- Audit/History/TTL-Legacy-Reference/：早期设计参考，非当前 Authority。
- community-research/：untracked，.gitignore:22，0 files tracked。
- historical L1 logs（60 篇，截至 2026-11-16）。

### GHOST / SPEC-REFERENCED BUT NOT PRESENT
- query.py / build_graph.py / MAP.yaml / golden-set infrastructure。（SessionStart hook 已移出：生成器卷落地，接线待活体验证；MAP 显式除外。atlas_fill 已落地为卷，见 corr-95，不再 GHOST。）
- **L1 自动装配 + recency 时间加权**（Owner 2026-11-22 裁按 GHOST 登记，corr-102）：SPEC-READ §装配预算表宣称「L1 ≤1K token 近 7 天相关日志（recency 半衰期 7 天加权）」、SPEC-WRITE「30 天降权」、SPEC-GOVERN「recency 衰减检索权重」——**功能整体无实现载体**：read_orchestrator `assemble()` 无 L1 段（预算表 `L1:1000` 为未消费死数字），session_start 不触 04-Logs，检索侧无任何时间加权。独立机制审计（MECHANISM-ADOPTION-AUDIT-2026-11-22 §6 判定①）实证；**规格文本已就地 GHOST 标记**（三处，Owner 2026-11-24 裁提前于 Beta 轮，corr-106），缺口以本台账显式登记。
- **Do not create these merely because Specs mention them.**

依据：PHASE0-INVENTORY（脚本地图原本）；卷目录 script-volumes/（11 卷，+atlas_fill 导航刀 G corr-95，+fed_sync 联邦三方合并 corr-119）。

## 5. 阶段边界

- **P0 COMPLETE**（offline↔CLI reconciliation / HEAD-index convention / P0-c corruption exercise PASS；详见 log-2026-09-29-06 #3）。
- **P1 first slice DONE**（one-hop + explainable hit path，read_orchestrator 卷，self-test 10/10）；two-hop PAUSED（superseded by convergence-first）；Leiden/embedding/community compilation = Not yet。
- **Scope freeze（逐字，绑定约束）**：

Until the current stage is accepted, do not independently:
- change the L1/L2/L3 model;
- create a new knowledge authority;
- add a database/vector store;
- add a new third-party execution entity;
- broadly rewrite the toolchain;
- implement P4;
- build infrastructure solely for future extensibility;
- treat historical snapshots as active rules.

- **Scope freeze 显式豁免（Owner 2026-11-24 裁，Beta 提案 r4 裁②，corr-107）**：GitHub Actions CI 获准作为 B4 机械化载体——仅跑既有只读工具链（vol_extract→entry_doctor→--self-test，三平台矩阵），不新增知识权威、不写库、不改治理模型；此为「add a new third-party execution entity」条款的唯一显式豁免，逐字登记，不扩及其他实体。

## 6. 已知结构问题

- **Document/history contamination**: 历史设计文档含旧访问模型快照；当前 runtime 真值取 Formal Specs，不取历史节。
- **MCP enforcement gap**: IDE 可能不强制 tool-level filtering；Agent 须守 read-only whitelist，不为便利扩权。
- **Atlas 派生投影决策**（Owner 2026-11-12 采纳；**2026-11-18 23:40 授权实现**——atlas_fill 刀G+SessionStart hook 刀H 已批，MAP.yaml 显式除外仍 GHOST；触发确认+实施依据=终版导航计划，收录判据待 Owner 裁后 --apply）。Atlas 卡片清单=domain 标签自动投影（派生视图），非手工平行清单。当前单 SH 不拆，G 落地前维持 Stage 0 手工态。
- **types.json**: 0 bytes tracked（ddd8f6c），2026-09 清空事故残留；保持跟踪不手写，待 Obsidian 重写。

依据：SPEC-READ / 00-INDEX / PHASE0-INVENTORY / corr-2026-11-12-36。

## 7. 悬置决策清单

- ~~联邦制推广波1 triage 标准~~ **已验收**（2026-11-18，corr-63）；T3 机械情态扫描 deferred（Goodhart 风险，Owner 未推翻）。
- 再结晶-分化条款（首次真实分化后定）。
- G-2 成熟度阶梯（deferred）。
- GPV_NAME 命名规则（DP-6 零改名，无可计算判据→不立规则）。
- **等真实触发（不造任务；原 Next task 尾段归位，corr-106）**：漂移实例累积挂靠表述纪律卡；首次真实再结晶-分化后立条款；推广波2（枢纽显式化）待自然聚簇；四分分类学成熟度升迁待真实独立使用；审核者 A/B 映射待首个真实独立审核任务（Owner：备用角色短期不开发）；reflect 收集器噪声过滤为未来工具候选；G2-6「其余角色分步收」候诊（corr-86，等新角色真实出现）；遗漏型漂移治理机制暂登记实例不立律（表述纪律卡实例节 corr-86 行）。
- **Beta 后候选批（r4 已裁归口，corr-107）**：L1 manifest（裁③不并入 Beta 门槛）、同步脚本卷化、多样化冷启动推广（裁④）、doctor「薄」可计算判据（留观察）、审计 §4.2 包装风险 caveat、inline-list 约定放弃、候选结晶批（攒 9 条：log-04/05/11-23-01 六条+log-11-24-02 两条+log-11-24-03 门面重心纪律一条）。CI 已裁批准（§5 豁免），不再候裁。

## 8. 更新规则

Update this file only when current project facts or scope have actually changed.

Do not turn STATUS into a design diary.

Do not encode an Agent proposal as a user-approved decision.

If a matter needs architectural/user judgment, mark it:

`NEEDS USER DECISION`

**BCP 变更协议（批3 补入）**：每次变更先落 L1（log/correction）再改本文件指针；历史叙述不入本文件，留指针指向原本。孤本事实（仅存本文件者）须先下沉 L1 再删。