---
id: "c-2026-00026"
layer: 2
type: concept
project: memory-lib
domain: [knowledge-engineering, memory-governance]
maturity: seed
provenance: reflect
created: 2026-11-16
last-touched: 2026-11-16
summary: 自指信任根与TCB最小化：sha与代码同文件自证无外部锚（Thompson信任的信任）；删装饰性可伪造真值通道优于加固；强制self-test前置=POST；外部锚=git+人类裁判
k: []
v: []
e:
  - { target: "[[SH-knowledge-engineering]]", rel: part_of, weight: 0.7 }
  - { target: "[[EFG三原理]]", rel: implements, weight: 0.7 }
  - { target: "[[根因聚类与结构性修复]]", rel: co_occurs, weight: 0.6 }
  - { target: "[[entry_doctor校验清单与自检法]]", rel: co_occurs, weight: 0.5 }
aliases: [自指信任根, TCB最小化, 信任的信任]
---

# 自指信任根与 TCB 最小化

## 定义

校验系统的两个结构性命题：**完整性锚不能与被保护物同处自证**（信任根问题），以及**可信计算基（TCB）应最小化**——删掉装饰性、可伪造的真值通道，优于加固它。源自 Thompson《Reflections on Trusting Trust》（1984 图灵奖演讲）的 TTL 落地。

## 核心

reflect 编译自 doctor v2.7.0 TCB 收缩（来源：corr-2026-11-16-55、红队 S1/S9/S10）：

### 1. 自指盲区（Thompson 信任的信任）

sha256 与被保护代码块**同处一个文件、由同一作者声明**——能改库者就能改 doctor 并重算 sha，使整条校验链信任一个被阉割的校验器。这是 Thompson 编译器自指攻击的同构：被篡改的编译器编译出被篡改的编译器，自证不可能。**完整性锚必须锚定在外部**。

### 2. 外部锚的务实选择（不做过度工程）

单用户私有库场景，外部锚**已经存在**：git 历史 + Owner 裁判。这正是哲学保留人类裁判的深意。便宜闭合：

- **强制 self-test 前置（POST 式）**：信任一次扫描前先跑 `--self-test` rc=0。self-test 是真破坏性探针（阉割版必 MISS），强制前置即拦住「阉割版 doctor 无需被检出即放行」。
- **doctor 卷改动必附 correction**：变更留审计痕迹。
- **拒绝外置签名基础设施**：对当前威胁模型是过度工程（TCB 膨胀）。

### 3. TCB 最小化：删 > 加固

**给装饰性数字建可伪造的真值通道，本身就是 TCB 膨胀**。实证：RULES/SELFTEST 计数锚点——A 评审判「装饰」，B 红队证明真值可伪造（注释伪 Rule 行/假 [OK] 行）。两个独立评估从不同方向指向同一处 → **删掉锚点（缩 TCB）优于加固真值取法**。删后文档不再宣称具体计数，无宣称即无对账需求。「最好的校验是不需要校验。」

### 4. 真值通道若保留，须结构化不可伪造

若将来恢复计数对账，真值须用结构化输出（ast 解析注册表/退出码/断言通过数），而非可伪造的文本行计数（数 `[OK]` 前缀行可被 print 假绿）。

## 存疑

- 强制 self-test 前置是纪律（SPEC-GOVERN），非代码强制——执行者仍是 Agent 自觉，E 原理意义上未完全闭合（覆盖图记接受风险）。
- git+Owner 外部锚对「能改库者」有效，对「能改 git 历史者」无效——威胁模型边界须明示。
- TCB 最小化与覆盖度有张力：删通道缩攻击面，但也少了探针；判据=该通道是否承重（VERSION 承重保留，计数装饰删除）。

## 应用

设计任何「校验器/完整性锚/真值通道」时：先问锚是否自指、通道是否承重、真值是否可伪造。实证：doctor 删 RULES/SELFTEST 锚点 + 强制 self-test 前置，S9/S10 攻击面消失，S1 由纪律闭合。相关：[[EFG三原理]]（E 契约需执行者/F 独立裁判）、[[根因聚类与结构性修复]]（RC2 根因）、[[entry_doctor校验清单与自检法]]（self-test 破坏性探针）。