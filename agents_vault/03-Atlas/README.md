---
id: ov-2026-readme
type: overview
created: 2026-11-20
last-touched: 2026-11-24
summary: TTL 项目全景地图（纯人读）：是什么/设计取向/运作/拓扑/Clone&Run/权威地图/出处；非定义非法律，Agent 执行任务不加载本件
aliases: [Project Map, 项目地图]
---

# TTL — 项目全景地图

> ## NON-AUTHORITY / PROJECT MAP / 纯人读
>
> 本件是**地图（map），不是定义（definition），更不是法律（Runtime Specs）**。它不授权任何 runtime 变更，不替代：`AGENTS.md` · [[05-Governance/00-ENTRY|00-ENTRY]] · [[SPEC-READ]] / [[SPEC-WRITE]] / [[SPEC-GOVERN]] · `90-Meta/schema.yaml` · [[00-ROOT]]。与任何 Authority 冲突时，**Authority 胜出**。
>
> **读者=Owner / 使用者（人）**。Agent 执行任务不读本件：本件 type: overview（第四类别，schema `overview` 节），不进检索装配（read_orchestrator 类别过滤执行）、不进 ROOT L0 / SH-index 角色跳转。Agent 的身份与义务入口在 [[SH-index]]。
>
> **本件同时是项目理念内容的当前态权威宿主**（2026-11-20 Owner 裁：理念归档入库，Audit 三份解读源降格为出处）。这不改变其 NON-AUTHORITY 性质——理念是设计意图的表述，runtime 行为仍以正式 Spec 为准。

> **库外文档说明**：本件各处（§1/§5/§6/§8）提到的 `Audit/` 路径（总定义草案、核心假设规格、证据仓库、历史参考、理念解读源）均在**私有工作区，不随公开库发布**——其中含协作现场痕迹（本机路径、会话上下文），整理公开的成本大于收益；项目开发历程已由库内 `04-Logs/`（日志+更正链）完整承载。公开克隆者看不到这些文件属预期，不是断链；引用它们处均为出处注记，不影响库内运行。如需查阅，欢迎通过 GitHub 联系项目维护者。

---

## 1. 项目是什么

TTL 的目标不是「让 AI 记住东西」，而是**让 AI 产出的知识可被信任**。核心命题：AI 的语义收敛**能被治理到可信**。可信度的依据不是模型输出的确定语气，而是一套纪律：**保留不确定、保留来源、保留人类裁判**。

> **对外名映射（Owner 2026-11-22 裁）**：对外名 **Stratified Governed Convergence**（SGC），内部代号 **TTL**（立项双关代号：Time to Live + Through the Lens，非缩写，与现理念关联已淡）。三名各承一处哲学：**Stratified**=知识生命周期分层（L1 不可变情景 → L2 可变语义 → L3 权威规格，每层修改契约不同，沉积式底座约束上层）；**Governed**=治理即自然语言文本、分权与人类裁判（治理与知识同库同格式，非 DB/代码形态）；**Convergence**=AI 语义收敛为核心假设。历史文件/日志/卡片保留 TTL 代号不改写（L1 append-only）。repo 名 `stratified-governed-convergence`。改名原因与命名裁断链见根 README 命名节 / D35 / corr-97。

- **Maturity = alpha**（Owner 2026-11-14 宣告）。**Alpha = 确定性收束成功**：系统担保收敛过程**确定、诚实、可复现、可审计**（该收束的收束、该存疑的存疑），**不担保每条结论正确**。正确性是研究假设，用 Gold 去*测量*、由 Owner 裁判，不是 Alpha 的担保物。【设计决定：读法 A，Owner 2026-11-12 裁（原 C-2）；读法 B（诚实可用里程碑即 Alpha）已被否决】
- **Markdown / Obsidian = 持久知识基底**：vault + Git 是知识与版本权威，文件是 source of truth；Obsidian 提供存储、链接引擎（`[[X]]` → 反链/图谱/search）、人类界面。
- **AI 语义收敛 = 核心假设（验证中）**：AI 读原始 L1 记忆，做 interpretation / correspondence / conflict detection / abstraction / consolidation，收敛为稳定 L2/L3 知识。**总体 PARTIALLY VERIFIED——待验证假设，不是已证明事实**；不得把单次 dogfooding 证据升级成普遍规律。
- **确定性代码 = 语义层的支撑结构**：提供 identity / structure / routing / invariants / permissions / provenance / persistence / rollback / validation 九项骨架，**不是知识本体**。
- **backend / search / index 可替换**：TTL 的核心是语义责任 + 治理协议，不是某个特定实现。

系统总定义草案（DRAFT / AUTHORITY CANDIDATE）与核心假设可检验化规格仍在库外：`Audit/Current/TTL-SYSTEM-SPEC-DRAFT.md`、`Audit/Current/TTL-Core-Hypothesis-Spec-NON-AUTHORITY/SEMANTIC-CONVERGENCE-CORE-SPEC.md`（升格须 Owner 显式决策 + 移入库内）。

## 2. 工作纪律与自指验证

基本纪律：**记录「不知道」优先于给出无依据的「知道」**。三条根命题：**能力不蕴含许可，存在不蕴含权威，提案不蕴含决定**。操作形态见 [[表述精确性纪律]]（表述与实物一致）与 [[不越权纪律]]（行为与授权一致）。

**自指验证**：项目的开发形态（Agent 执行 + 确定性骨架约束 + 人类裁决）与 TTL 要验证的命题同构，开发过程本身即核心假设的测试场景。若开发需要 Owner 深度介入才能收敛，构成关于核心假设的负面证据；反之构成正面证据。开发中出现的漂移、泄露、探针缺陷均作为假设相关证据记录（证据流：`Audit/Current/TTL-GOVERNANCE-DOGFOODING-LOG.md`，正面与负面样本并存，不作选择性保留）。

**日常运作一览**（两条正交导航轴 + 一条变更流）：

- **知识轴**：[[00-ROOT]] 是唯一根入口，L0 常驻层（运行模式/访问通道/分区一览）先读，再按需展开规格书→域入口（SH-{domain}）→卡片——渐进披露，多数查询只触达浅层，不盲扫全库。
- **身份轴**：Agent 先经 [[SH-index]] 认身份（维护者/消费者/联邦/审核者四分），只读对应角色件的义务指针，不加载无关域清单——权限差异由路径承载而非事后检查。新立项/上下文压缩后的首个动作即走此入口。
- **变更流**：工作 Agent 提交增改+申报报告 → doctor 闸门校验 → 并入；重大裁决归 Owner。每次实质产出经守卫式原语追加一篇日志（L1 append-only），错了新开更正记录回指——当前态不是快照，而是可追溯的变更链。当前状态唯一指针见 [[STATUS]]。

## 3. 设计取向

> 以下取向源自项目初衷，仍在演进，**有轻重缓急，但不固化为教条**——不定「支柱」编号、不排核心权重（Owner 2026-11-12 明示）。每条标注权威状态：【设计决定】=Owner 明示裁决；【Owner 拾起】=明示拾起的取向，未逐条立律；【待校准】=Agent 解读（INFERENCE），升格须 Owner 明示，在此之前不得当作已裁决的设计决定引用。出处锚点见 §8。

### 3.1 基础取向（2026-11-12 解读）

1. **认知诚实——「不知道」是一等公民**。证据不足必须保留存疑，冲突未解决必须保留双方，不得为了收敛而静默消除不确定（Anti-Convergence）。多数记忆系统优化「给出答案」，TTL 优化「不给出假的确定性」。其推论**「重在确定性收束，而非确保无误」**为 Owner 2026-11-12 明示【设计决定】；取向本身【待校准】。
2. **分权——语义、骨架、权威职责分离**。两条铁律：代码不得替 AI 猜语义关系；语义责任 ≠ 实现载体。AI 不得自我认证；Owner 裁决不可由事实陈述推定（事实陈述不构成授权）。权力制衡落在**权威形态**而非体量：治理规则零行代码承载、代码无常驻权威（全部卷化入图、可更正）——实测代码行数反而多于治理文本（约 1.5 倍，2026-11-22 独立审计复算；早期「治理文本体量大」表述系修辞残留，已更正，corr-103）。【待校准；铁律原文锚点=总规格书草案】
3. **反物化——表征不等于事物本身**。三个不可合并的区分：**检索 ≠ 语义对应、图边 ≠ 语义关系、存储 ≠ 记忆生命周期**。存储的链接不等于已判断的语义关系，持久化的字节不等于记忆生命周期；README 是地图不是定义。适用域已扩至三层（2026-11-16 补遗）：**载体层**（源与产物分离：.md 唯一源，.py 临时副产物用完即删——常驻产物=第二份手抄源）、**数字层**（凡能从单一权威派生的数字，手抄进 prose 就是物化）、**分类层**（盘点结果不落盘为逐文件标记——27 个标记即 27 个漂移点）。可操作判据：**该表征是否具有独立于权威的第二份维护责任——有则存在物化风险**。【扩展待校准（原 C-4）；三层落地均有 corr 实证】
4. **可证伪——把自己的核心主张当成可能错的**。核心假设显式冻结成可检验规格，配盲态、Gold、机械评分、失败归因。「执行完成 ≠ 假设通过」。建仪器是为了**发现自己错**，不是确认自己对。【待校准】
5. **知识是有来源的、不可篡改的历史**。L1 append-only，更正只能新开日志回指；Git 是版本权威；hash 封存用于实验记录的完整性存证。**当前态不是单一快照，而是可追溯的变更链。**【待校准；append-only/Git 机制为库内规格 FACT】
6. **语义约束为引擎，确定性骨架做薄**。【Owner 拾起：2026-11-12 明示重新拾起，此前曾被战术性放弃】三层含义：①约束的主要载体是自然语言——契约/Spec/治理文本是「给 LLM 读的规则」，权威在文本不在代码（代码体量可以更大，见 §3.1-2 更正），这不是冗余；②预期 LLM 理解力随迭代提升——这是动态假设，须被验证而非默认成立，验证方式=软测试、分析子 Agent 思考过程（D8，非硬门，不做新旧模型差异测试）；③确定性骨架刻意做薄——doctor/reducer/hash 只做可机械核验的护栏，不做语义判断。**若后续开发把语义规则不断硬化成脚本，就偏离了这个取向。**「薄」的判据（2026-11-16 补遗）：**薄在表面积，强在语义**——只校验有外部锚的真值，装饰性真值通道删除优于加固（删=缩 TCB）；不在错误的抽象层上堆复杂度（用正则解析非正则文法导致修补无穷回归）；**薄不等于放行**——豁免必配替代判据（G 原理）。【判据待校准】

### 3.2 迁移重构轮补遗（2026-11-16）

7. **链接携带意图**（Owner 2026-11-15 点拨，G20 机制落地）：当前态引用写成 `[[目标#VERSION|别名]]` 活链接，历史归属保持纯文本——**意图由「作者写链接」这个动作本身表达**，零额外标记、零清单维护。让表征的结构差异承载语义区分，而不是另立元数据层去标注语义。【Owner 点拨+机制已落地】
8. **不妥协纪律——内部检查点 ≠ 交付边界**（Owner 2026-11-16 裁）：「先试点/浅起步」只能当内部验证检查点，不是交付边界。克制有两个方向的误用：该完整时以「小步」为名缩交付，与该谨慎时以「效率」为名跳闸门。**诚实的完整交付与诚实的存疑标注并行不悖。**【设计决定】
9. **自包含规划作为跨压缩连续性机制**：上下文压缩后，新上下文只读规划文件即可接续，不依赖会话记忆——**Agent 的连续性不靠记忆靠权威文档**，与「知识不靠物化靠活链接」同构。操作形态已入规格：新立项先走 SH 入口（[[SPEC-READ]] 入口规则第 6 条，2026-11-20 立；同日晚些时候扩至每次压缩后首个动作，corr-90——后者是本轮后续发展，非本源条目内容）。【条目归属待 Owner 裁（原 C-5）；机制多轮实证】
10. **字节保真的诚实边界**：Markdown 代码块语法本身不保证字节保真；本库的保真是**二进制抽取操作的功劳，不是 Markdown 给的**；结论不普适于任意脚本。**收益归因要诚实，适用边界要显式**——与「把 provisional 说成 certified」互为镜像。【待校准】
11. **接入与治理是两个独立承诺**。【设计决定：Owner 2026-11-16 裁升格（原 C-8）】统一接口降低的是*接入成本*（工具/索引/链接复用），不降低*治理成本*（每类本体仍需自己的可信判据）。**统一表征（皆文件）不等于统一本体（皆知识），更不等于统一可信（皆受治理）**——三个「统一」分开承诺、分开兑现。它是「存在不蕴含权威」在载体层的直接实例；内化翻译判据：**去掉来源词源后，命题在本体系内仍然自足成立**（Unix「一切皆文件」仅作历史参照，本库表述不依赖其词源）。操作形态：[[异构节点入统一文件图的治理范式]]。

### 3.3 治理拓扑轮补遗（2026-11-20）

12. **身份×知识双轴正交**：导航拓扑有两条正交轴——**知识轴**（什么知识在哪：域→SH 域入口→卡）与**身份轴**（你是谁该读什么能写什么：ROOT 角色轴→[[SH-index]]→角色件→义务指针）。两轴在 SH 交汇但互不污染：角色件只装指针不装真值，真值仍在规格。这是分权取向在导航层的落地——权限差异由路径承载而非事后检查；也是反物化在身份层的实例（义务包是指向义务的导航，不是义务的第二份副本）。【待校准（原 C-9）；落地=D32 裁决包（Owner 2026-11-20 批），盲测三跳零漂移实证】
13. **行政划分 vs 统计聚类**：本库的域/枢纽分组是**治理裁量分组**（Owner 裁量的行政划分），不是模块度算法的**统计聚类**。行政划分的边界由 Owner 裁决承载语义，统计聚类的边界由算法承载相似性——把统计聚类直接当域，是把相似性误当语义。**SH 是 link-only 导航根，不承载域内卡片内容的有损摘要**；「社区摘要」的功能对应物是卡片 frontmatter summary（≤50 token），内容压缩发生在卡摘要层不在 SH 层。算法可以**提议**聚类（作为事实），语义命名与边界裁决归 Owner。【待校准（原 C-9）；Owner 对 GraphRAG 类比系**批判性认可**——仅「SH=社区摘要」映射被反驳，其余结构类比认可（corr-83/85）】
14. **斩链原理**：转述链漂移（复述链复用误差逐级放大）的治理=**引文块强制回原文接触实物**——修订/取代/据以执行前篇承重断言时，嵌引用块承载原文小句+指针再写新表述，把转述链重置为「原文→新文」，误差从积分降为单次。这是「真相是链不是快照」在文本传承上的应用。执行者=Agent 自律，**无机械闸门**（机械情态扫描缓议—— STATUS 悬置清单 T3 项，Goodhart 风险，Owner 显式接受已知漏）——自律层会漏是设计内已知风险，非机制失败。【待校准（原 C-9）；引文纪律已落地（05-Governance/03-CHANGE-DISCIPLINE，corr-63），触发实录含拦截与漏网两态】
15. **遗漏型漂移——斩链的覆盖边界**：斩链治**转述失真**（被转述的内容保真），不治**转述遗漏**（该被携带的内容没被转述）——遗漏发生在选择函数里不在传输函数里。现有对账机制全是**共时性校验**（此刻文本 vs 此刻实物），缺**历时性校验**（过去写下的承诺 vs 现在的实现）。两亚型：**执行遗漏**（已判决未执行，污染当前态真值）与**登记遗漏**（合法候诊项被近义任务顶替后除名），闸门位置不同（编译覆盖对账 vs 语义完成判定）。认知诚实不仅要求不输出无依据的确定性，还要求不遗失已登记的未决事项。【待校准（原 C-9）；Owner 裁暂登记实例不立律，机制设想留 Owner（corr-86）】

### 3.4 已识别偏离形态与结构护栏（给参与开发的 Agent）

已识别的偏离形态：放松判据以使结果通过（工程师倾向）、把 provisional 表述为 certified（LLM 倾向）、因方案优雅而扩 scope 或把提案当决定（Agent 倾向）、跳过盲态或自签 attestation（效率倾向）、向 Owner 报告未经证实的乐观结论（迎合倾向）、把语义规则硬化成脚本（过度工程倾向）、把「当前读不稳」当永久结论或把「以后会读懂」当默认成立（静态假设倾向）。**约束由结构承载，不依赖 Agent 自律**：doctor 闸门、hash 封存、仪器冻结、Owner 闸门、报告分级。命中任一形态即停止，按 [[04-REPORTING]] 标 `NEEDS USER DECISION`。

## 4. 架构拓扑全景

TTL 的拓扑不是单一知识图谱，而是**分层路由树 + 带类型语义图 + 不可变事件底座的复合结构，叠加分权治理层**。下列工程范式（分布式系统、编译器、数据库、开源治理）是**事后解释类比，不是设计来源**——结构同构深浅不一（Event Sourcing/缓存层级较实，circuit breaker/QoS 等偏修辞标签），读作全景修辞而非严谨架构论证（2026-11-22 独立审计 §4.2 提示，corr-103）：

- **分层路由树**（缓存层级/局部性原理）：ROOT L0 → 分入口 → 卡 summary → 全文，子路径优先；多数查询只触达浅层，深层按需展开；L0 常驻（对应热缓存层），锚点只放指针不放副本。**身份轴**（D32）与知识轴并行：[[SH-index]] 认身份→角色件义务包→域入口，两轴正交（§3.3 第 12 条）。
- **装配预算**（QoS 分级+优雅降级）：硬上限+裁剪序 L1 先裁→L2 按相关度→L3 不裁；control plane（红线/路由）永不丢弃，best-effort 数据 plane 先牺牲。预算真值=schema `budgets` 节。
- **带类型语义图**（property graph / 编译器 typed IR）：`k=父 v=子` 分类层级 + `e={target,rel,weight}` 类型边 + rel 枚举；`grounds`（规格→证据）=proof-carrying 可追溯性。**嵌套单轨**：`[[X]]` 嵌在边里复用 Obsidian 原生反链/图谱/search，不另建图引擎——语义关系以原生链接为载体；`aliases` 取代 same_as，同义归一在映射层解决，不修改节点本体。
- **不可变事件底座**（Event Sourcing + CQRS / WAL / Git）：L1 append-only + corrections 回指 + reflect-to 派生指针；L2 可变视图与 L1 不可变源在拓扑上彻底分离；更正是显式边而非覆盖。
- **完整性多维**（图连通性/dead-code/GC 可达性）：orphan / broken-link / unreachable / stale 四分正交——有边 ≠ 可达 ≠ 新鲜；导航枢纽与原子卡有不同连通契约；纯人读地图（本件）豁免可达性但断链仍判。
- **多通道等价**（service discovery / circuit breaker / conformance testing）：探活→路由→降级；离线兜底须复刻原生语义、不得更能读；退离线须 Owner 批准（D33）；CLI 须证明 vault 归属。
- **分权治理**（maintainer / PR review / CI gate 模式）：branch→审核=并入 Agent→main，doctor 作 merge 闸门；权威是有向无环层级而非网状平权；提案→决定→执行单向流，节点不得自我升格。
- **operating-mode**（risk-based auth / 自适应 QoS）：由「可逆性+真安全+意图明确度」派生思考力度/确认模式；真安全绝对、谨慎性分级——把「谨慎」建模为可计算旋钮而非恒定美德。

综合：**语义引擎是 LLM 对自然语言契约的理解与迭代，不是脚本**——上述工程范式对应的是护栏与拓扑，不是语义引擎本身。当前入口与名册真值：[[00-ROOT]]（根+路由）、[[SH-index]]（身份根）、各域 SH 域入口（本件不复制清单）。

## 5. Clone & Run

**离线核心零第三方依赖**：runtime/辅助脚本只用 Python 标准库，**全部已卷化**——唯一源是 `90-Meta/script-volumes/*.md` 脚本卷（卷数与名单以目录为真值），`.py` 用时抽取、用完即删。**Python 3.11+** 全功能（TOML 配置解析需 `tomllib`）；3.10 及以下不用 TOML 配置时亦可运行。无需 pip install、无需 venv。**不引入 uv 是决定不是遗漏**（第三方 import 为 0、冻结 reducer 跨解释器逐字节相同、无环境可装；唯一引入触发条件=出现第一个真实第三方依赖）。

```powershell
git clone <repo-url> TTL
cd TTL
# 第 0 步：自举抽取器（卷→.py；卷化脚本的 .py 无常驻文件）
python -c "d=open(r'agents_vault/90-Meta/script-volumes/vol_extract.md','rb').read();o=b'````python\r\n';c=b'\r\n````\r\n';s=d.index(o)+len(o);e=d.rindex(c);open(r'agents_vault/90-Meta/scripts/vol_extract.py','wb').write(d[s:e])"
# 递归抽取全部卷（depends-on 拓扑序 + sha256 对账）
python agents_vault\90-Meta\scripts\vol_extract.py --all --vault agents_vault
# 结构闸门（应输出 PASS: 0 ERROR / 0 WARN / 0 INFO）
python agents_vault\90-Meta\scripts\entry_doctor.py --vault agents_vault
# 各卷自检（有自检的卷逐一 --self-test，应各 exit=0；read_audit 无自检）
python agents_vault\90-Meta\scripts\entry_doctor.py --self-test
# 只读检索探针
python agents_vault\90-Meta\scripts\read_orchestrator.py --probe "doctor" --vault agents_vault --channel offline --json
```

bash（Linux/macOS）等价配方——卷字节是 CRLF（仓根 `.gitattributes` 已标 `-text` 保字节，clone 后勿自行转换换行符）：

```bash
git clone <repo-url> TTL
cd TTL
# 第 0 步：自举抽取器（单引号包 python -c，\r\n 由 Python 字节字面量解释）
python3 -c 'import os;d=open("agents_vault/90-Meta/script-volumes/vol_extract.md","rb").read();o=b"````python\r\n";c=b"\r\n````\r\n";s=d.index(o)+len(o);e=d.rindex(c);os.makedirs("agents_vault/90-Meta/scripts",exist_ok=True);open("agents_vault/90-Meta/scripts/vol_extract.py","wb").write(d[s:e])'
python3 agents_vault/90-Meta/scripts/vol_extract.py --all --vault agents_vault
python3 agents_vault/90-Meta/scripts/entry_doctor.py --vault agents_vault
python3 agents_vault/90-Meta/scripts/entry_doctor.py --self-test
python3 agents_vault/90-Meta/scripts/read_orchestrator.py --probe "doctor" --vault agents_vault --channel offline --json
```

- **工具版本真值=各卷代码块 VERSION 常量**（本件不镜像版本号；当前态版本宣称见 [[STATUS]]）。
- **vault 定位**：`--vault` → `TTL_VAULT` 环境变量 → 用户 TOML 配置 → 从 cwd 向上发现 → 脚本在 vault 内时 legacy 兜底。
- **克隆不含**（.gitignore 刻意排除）：`.venv/`、`agents_vault/.obsidian/plugins/`（含 REST 插件密钥——密钥不入库）、`community-research/`（第三方快照）。
- **完整体验（可选）**：用 Obsidian 打开 `agents_vault/`；MCP/REST 读取通道自行安装 Local REST API 插件并生成自己的 API key。写入主通道=官方 `obsidian` CLI + App 运行。
- **路径与换行符无关性已实测**：全新 clone 到任意路径后闸门/自检/探针/定位全通过（Windows 实测），tracked 代码无硬编码绝对路径；仓根 `.gitattributes` 标脚本卷 `-text`，Windows（autocrlf=true）与 Linux/macOS（autocrlf=input）双态 clone 实测卷字节保真、抽取+doctor 全绿（2026-11-22，独立评估 V11 回归）。**双态实测范围=卷字节保真+抽取+doctor**；`--self-test` 当时仅 Windows 实测——POSIX 上 CLI_VAULT_PIN 的 Windows 盘符 fixture 必失败（外部 Linux 评审发现，v2.16.1 平台门控修复+POSIX 模拟验证，真 Linux 重验待外部确认，corr-106）。

## 6. Authority Map

| 层 | 路径 | Authority | 管什么 |
|----|------|:---------:|--------|
| Agent 入口 | `AGENTS.md`（仓库根） | **YES** | 工作纪律：read state → single task → minimal change → verify → report |
| 治理 | `05-Governance/`（含 [[STATUS]] · [[05-Governance/00-ENTRY|00-ENTRY]] · 01~08） | **YES** | 项目状态、工作边界、stage gate、变更/报告纪律、恢复策略 |
| 运行时规格 | `01-Specs/`（[[SPEC-READ]] · [[SPEC-WRITE]] · [[SPEC-GOVERN]]） | **YES** | runtime 怎么读/写/治理 |
| frontmatter 权威 | `90-Meta/schema.yaml` | **YES** | 字段/类型/约束唯一权威 |
| 知识根入口 | [[00-ROOT]] | **YES** | 唯一根 + 路由（L0 渐进披露） |
| 理念当前态宿主 | 本件（`03-Atlas/README.md`） | **理念内容 YES / runtime NO** | 设计取向与拓扑意图的当前态表述；不定义 runtime、不授权变更 |
| 系统总定义（草案） | `Audit/Current/TTL-SYSTEM-SPEC-DRAFT.md` | **NO — DRAFT / AUTHORITY CANDIDATE** | 架构/语义整合草案；升格须 Owner 显式决策+移入库内 |
| 核心假设规格 | `Audit/Current/TTL-Core-Hypothesis-Spec-NON-AUTHORITY/SEMANTIC-CONVERGENCE-CORE-SPEC.md` | **NO — CORE-HYPOTHESIS** | 语义收敛假设的可检验化定义 |
| 证据仓库 | `Audit/`（导航 `Audit/00-INDEX.md`） | **NO — NON-AUTHORITY** | 审计/研究/交接证据；理念解读源已降格为出处（§8） |
| 历史参考 | `Audit/History/` | **NO — HISTORY / REFERENCE** | 保留早期设计与研究，不左右当前判断 |

> **绝不**从 Git 存在、工作区存在或文件名推断 Authority。任何 Audit 材料升格为 Authority，须经 Owner 显式决策 + 移出 Audit 写入 `01-Specs/` 或 `05-Governance/`。

## 7. 当前状态

**唯一指针：[[STATUS]]（gov-status）**——maturity/stage/next task/工具版本/计数/悬置决策均以该件当前态为准，本件不镜像阶段叙述与可派生计数（反物化数字层纪律）。历史裁决链：`04-Logs/corrections/` + `04-Logs/logs/`。

## 8. 出处与迁移注记

**理念内容出处锚点**（本件 §2~§4 由下列三份源理解重写而来——重写行文组织，术语/命题内涵/权威状态/出处锚点四者保真；源文件保留于 Audit 作出处，正文不动）：

| 本件条目 | 源 | 源内位置 |
|---|---|---|
| §1 本质/Alpha 定义 | 主解读 | §0 + §6 C-2（读法 A 已决） |
| §2 工作纪律/自指 | 主解读 | §2 + §5 |
| §3.1 六条基础取向 | 主解读 | §1（末条 Owner 拾起）+ §6 C-1 |
| §3.1 反物化三层扩展/薄判据 | 补遗二 | §1 + §2（C-4 待校准） |
| §3.2 第 7~10 条 | 补遗二 | §5.1~§5.4（第 9 条归属待裁=C-5；第 7/8/10 条随 C-4 整体待校准） |
| §3.2 第 11 条（接入与治理） | 补遗二 | §5.5 + §6 C-8（已升格设计决定） |
| §3.3 第 12~15 条（双轴/行政vs统计/斩链/遗漏边界） | 补遗三 | §1~§4（C-9 待校准） |
| §3.4 偏离形态与护栏 | 主解读 | §4 |
| §4 拓扑全景 | 主解读 §3 十行表 + 身份轴（D32 裁决包） | §3 + §3.1 |
| §5 Clone & Run / §6 Authority Map | 库外原 README | §0 / §2（路径按库内真值核正） |

三份源（NON-AUTHORITY / INTERPRETATION，降格后作出处；库外文档，纯文本引用）：
- `Audit/Current/TTL-PROJECT-PHILOSOPHY-INTERPRETATION.md`（2026-11-12 主解读）
- `Audit/Current/TTL-PHILOSOPHY-SUPPLEMENT-2026-11-16.md`（2026-11-16 迁移重构轮补遗）
- `Audit/Current/overview-internalization-2026-11-20/PHILOSOPHY-SUPPLEMENT-3-governance-topology-2026-11-20.md`（2026-11-20 治理拓扑轮补遗）

**权威状态迁移说明**：源内已决条目（读法 A、接入与治理两个独立承诺、不妥协纪律等）的设计决定地位**随内容迁入本件，不因源宿主降格而失效**；待校准条目（C-1/C-4/C-5/C-9 等）保持待校准，不因重写而升格。校准编号沿用源内原编号以便回指。

**本轮迁移安排**（overview 内迁轮，Owner 2026-11-20 批准；执行进度见 [[STATUS]]）：
- OV-3：库外原 README（重要历史文件）完整保存至 `Audit/History/TTL-Legacy-README/`（目录内附本体声明）；仓库根 GitHub 门面 README 改为一行仓库相对路径指针指本件。
- OV-4：三份源顶部加迁移注记降格为出处（正文零改动，corr-65 时点注记先例）；理念源引用盘点（当前态活文件顶部注记，历史记述不改）。

---

*本件是 NON-AUTHORITY / PROJECT MAP（理念内容当前态宿主）。它不创建平行文档体系、不授权任何 runtime change。定义见总规格书草案，法律见 Runtime Specs，当前态见 STATUS。*