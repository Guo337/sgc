---
id: gov-2026-gov-index
type: reference
status: ACTIVE
created: 2026-11-16
last-touched: 2026-11-25
aliases: [治理族导航, Governance Index]
---

# 05-Governance 导航（治理族）

> 治理族 = Agent 行为约束与项目状态治理（原 `Agent-Governance/`，内迁完成 10/10）。**非知识条目**：不结晶、不升迁、不进检索装配；校验走 GOV_* 替代判据（schema `governance` 节）。分权靠 type + 01-AUTHORITY 两条 precedence，目录只是导航。
>
> **入口锚**：00-ENTRY 已内迁（批 4）；本页是治理族导航锚，GOV_ENTRY 判据已升 ERROR（可达性闭环，corr-2026-11-16-61）。

## 已内迁

- [[07-TERMS]] — 术语对照（type: reference，不绑定；批 1 试点，见 corr-2026-11-16-57）
- [[01-AUTHORITY]] — 权威与冲突规则（type: constraint，P0，self-referential；批 2，见 corr-2026-11-16-58）
- [[02-STAGE-GATE]] — 阶段门纪律（type: constraint，P0；批 2）
- [[03-CHANGE-DISCIPLINE]] — 变更纪律（type: constraint，P0；批 2）
- [[04-REPORTING]] — 汇报纪律（type: constraint，P0；批 2）
- [[05-SECURITY]] — 安全红线（type: constraint，P0；批 2）
- [[06-RECOVERY-STRATEGY]] — 收线策略（type: constraint，P0；批 2）

## 已内迁（续）

- [[STATUS]] — 项目当前态指针（type: status 单例 gov-status；批 3 先瘦后搬，见 corr-2026-11-16-60）
- [[00-ENTRY]] — Agent 入口协议（type: constraint，P0；批 4，见 corr-2026-11-16-61）
- [[08-BLIND-EXECUTION-ISOLATION]] — 盲态执行隔离（type: constraint，P1；批 4）

## 后续新增

- [[09-BETA-GATE]] — Beta 闸门判据（type: reference，FROZEN 公开件；Owner 2026-11-24 裁冻结，corr-121）

**内迁完成（10/10），旧路径已退役**：`Agent-Governance/` 目录已删除（丙案，Owner 裁，corr-2026-11-16-62）；历史文档中的旧路径是记述事实，追溯介质=git 历史（[[01-AUTHORITY]] §4.1）。新引用一律用本目录路径。

### 旧→新路径对照表（历史文本里的旧路径按此跳转）

| 旧路径（Agent-Governance/…） | 新路径（05-Governance/…） |
|---|---|
| 00-ENTRY.md | [[00-ENTRY]] |
| 01-AUTHORITY.md | [[01-AUTHORITY]] |
| 02-STAGE-GATE.md | [[02-STAGE-GATE]] |
| 03-CHANGE-DISCIPLINE.md | [[03-CHANGE-DISCIPLINE]] |
| 04-REPORTING.md | [[04-REPORTING]] |
| 05-SECURITY.md | [[05-SECURITY]] |
| 06-RECOVERY-STRATEGY.md | [[06-RECOVERY-STRATEGY]] |
| 07-TERMS.md | [[07-TERMS]] |
| 08-BLIND-EXECUTION-ISOLATION.md | [[08-BLIND-EXECUTION-ISOLATION]] |
| STATUS.md | [[STATUS]] |

> 本表是导航投影（零改名判例：文件名全部未变，仅目录前缀替换）；权威仍是库内本体，表本身不承载约束语义。

## 权威边界

- 治理族管「Agent 怎么干活」；[[SPEC-READ]] / [[SPEC-WRITE]] / [[SPEC-GOVERN]] 管「TTL 怎么运转」——两条 precedence 见 [[01-AUTHORITY]]（内迁后逐字保留，§1 路径表述已同步）。
- 治理族不进副库复制名单（federation: never）；副库 doctor 见 gov-* 判「非本库本体」不报错。