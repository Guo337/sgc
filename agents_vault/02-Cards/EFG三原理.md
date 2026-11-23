---
id: "c-2026-00012"
layer: 2
type: concept
project: memory-lib
domain: [knowledge-engineering, memory-governance]
maturity: growing
provenance: reflect
created: 2026-09-28
last-touched: 2026-11-20
summary: 规格工程三准则：E 契约需执行者、F 评测需独立裁判、G 豁免需替代判据（源自外部评审质询 E/F/G 拆解）
k: []
v: []
hub: true
e:
  - { target: "[[SH-knowledge-engineering]]", rel: part_of, weight: 0.7 }
  - { target: "[[SPEC-GOVERN]]", rel: grounds, weight: 0.6 }
  - { target: "[[entry_doctor校验清单与自检法]]", rel: co_occurs, weight: 0.5 }
  - { target: "[[entry_doctor]]", rel: co_occurs, weight: 0.5 }
  - { target: "[[异构节点入统一文件图的治理范式]]", rel: abstracts, weight: 0.7 }
  - { target: "[[独立评估与校验器独立性]]", rel: abstracts, weight: 0.7 }
  - { target: "[[自指信任根与TCB最小化]]", rel: abstracts, weight: 0.7 }
  - { target: "[[上提验收流程]]", rel: abstracts, weight: 0.7 }
  - { target: "[[entry_doctor校验清单与自检法]]", rel: abstracts, weight: 0.7 }
  - { target: "[[表述精确性纪律]]", rel: abstracts, weight: 0.6 }
  - { target: "[[不越权纪律]]", rel: abstracts, weight: 0.6 }
aliases: ["EFG三原理", "E/F/G 三原理", "规格工程三准则"]
---

# E/F/G 三原理（规格工程准则）

## 定义

规格/治理工程的三条通用准则，源自对外部评审质询 E/F/G 的拆解：**E 契约需执行者、F 评测需独立裁判、G 豁免需替代判据**。是本库治理模型与工具设计的共同理论根据。

## 核心

reflect 编译自候选结晶（来源日志：log-2026-09-27-03、log-2026-09-27-04）：

- **E — 契约需执行者**：光写规则不等于数据遵守；规格/契约若无**自动执行者**（校验工具），会在窗口期继续漂移（本项目曾漏 SH 语义矛盾为证）。落地：`entry_doctor` 作为可执行闸门。佐证：doctor 首跑即逮到自身误报 + 一条真实幽灵链接（log-2026-09-27-04）——证明"有执行者"才逼出隐藏漂移。**推论（工具先行于组织）**：引入多 Agent 治理角色前须先有可执行校验工具——没有 doctor，审核 Agent 无事可审、契约只是文档；演进顺序 Schema Profile → Minimal Doctor → Governance Roles。
  - **E 的两级失效（G22/G23 实证，2026-11-16）**：「契约需执行者」不止要求*机械执行者*，还要求*文本执行者*——义务须以**动作指令**写进权威文本，而非仅终态描述。两级失效：① **文本缺位**（更上游）：义务只活在注释/卡存疑节，Spec/ROOT 只有「.py 无常驻文件」终态描述无「用完即删」动作指令，Agent 读权威文本拿不到该步（G23，RC3 极端形态：契约连文本执行者都缺）；② **文本存在但纪律跳步**：规则 P0 ACTIVE 写明仍失守（G22 方向裁决误当执行批准；G23「不进 git≈合规」自我合理化）。**修法分层**：文本缺位→补动作指令（corr-56 修法 A，零代码可治）；纪律跳步→文本补全治不了，须机械执行者或显式标注接受风险（VOL_NO_PY 因误报被否，纪律层目前无解）。
  - **E 的自指陷阱（作者即违反者）**：编排方亲手把「用完即删」结晶进 00028 卡，跑完闸门即跳步——**「写进文档」被误当「已经治理」**。这是 E 在*认知层*的失效：表征（写了规则）≠ 事物（规则被遵守），恰是反物化在治理行为上的投影（「接入与治理两个独立承诺」，当前态表述见 03-Atlas/README.md §3.2 第 11 条；出处=2026-11-16 补遗 §5.5，已降格 REFERENCE）。**纪律**：每写一条纪律须同时问「执行者是谁？」若答案是「Agent 自觉」，就是在记录一个*已知会漏*的不变量，必须显式标注接受风险，不得假装已被治理。G22→G23 连续两次实证此模式。
- **F — 评测需独立裁判**：自出自标自评 = 自己判卷、循环论证。落地：治理模型**角色分离**（产出者 ≠ 批准并入者，见 [[SPEC-GOVERN]]）；golden-set 终审权留用户（F 的可靠性底线，不可自证）。
- **G — 豁免需替代判据**：豁免某项检查时，须另立**可计算的替代判据**，否则豁免=盲区。落地：stale 需可计算信号（被 supersedes 指向 / last-touched 超龄）且仅 L2 适用；Atlas 以 `edge-policy: exempt` 豁免 frontmatter 边后，改判"auto 区块非空 + 列出链接可解析 + ROOT 可达"。

## 存疑

- **F 只是部分独立**：审核 Agent 若与工作 Agent 同底座模型，会共享盲区；故 golden-set 终审留用户。
- **E 的执行者覆盖度有限**：doctor 离线正则是近似（断链/可达性），非完整链接引擎，复杂情形可能漏判。
- **E 的纪律层无机械解**：文本执行者可补（动作指令），但「文本存在仍跳步」目前无机械闸门可治（G22/G23）；凡执行者=「Agent 自觉」的不变量，本质是已知会漏的接受风险，须显式标注而非假装已治理。这是治理假设「治理文本能约束 Agent」尚未成立的直接证据（dogfooding log §3.1）。

## 实例（解压路径）

本枢纽抽象自以下实例卡（abstracts 边，implements 的逆；黑天鹅修订时顺此解压）：

- [[异构节点入统一文件图的治理范式]] — G 原理（豁免需替代判据）在脚本卷/治理族异构节点上的落地
- [[独立评估与校验器独立性]] — F 原理（评测需独立裁判）的三通道强隔离实证
- [[自指信任根与TCB最小化]] — E 原理（契约需执行者）+ F 原理在自指信任根上的落地
- [[上提验收流程]] — F 原理（角色分离：产出者≠验收者≠终审）的跨库流程化落地
- [[entry_doctor校验清单与自检法]] — E 原理（契约需执行者）的可执行闸门用法规范
- [[表述精确性纪律]] — E 原理「执行者=Agent 自觉须显式接受风险」在言语行为层的落地（corr-63）
- [[不越权纪律]] — E 原理「执行者=Agent 自觉须显式接受风险」在行为授权层的落地（corr-90/92；与表述纪律正交姊妹卡，彼治说的对不对此治做的许不许）

## 应用

贯穿本库：SPEC-GOVERN 的角色分离（F）、doctor 闸门（E）、stale 与 Atlas exempt 的替代判据（G）皆是 E/F/G 的具体落地。可迁移到任何"规格 + 校验 + 治理"系统设计。

## changelog

- 2026-11-18：maturity seed→growing（Owner 批 P-F，corr-2026-11-18-63）。证据：入度 5 磁盘实测（校验清单/上提验收/异构节点/独立评估/自指信任根均 implements 本卡），B6 升迁联动——被 N 卡 implements=成熟证据；三原理跨工具/流程/信任根/范式四类落地稳定。[VERIFIED: p_verify.py 2026-11-18 入度扫描]
- 2026-11-20：补 abstracts→[[不越权纪律]] 边+实例节行（Owner 裁 1A，corr-93）。诱因=导航波2/3 基线重算抓出：不越权卡 implements→本卡但本卡缺 abstracts 回边（建卡时漏加），SH 手工清单挂 EFG 组与规则失配；doctor HUB_DECOMPRESSION 单向查（abstracts→implements）漏网此反向形态。补边后互逆对账闭合，入度 5→6。
