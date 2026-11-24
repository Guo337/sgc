---
id: gov-2026-authority
type: constraint
status: ACTIVE
priority: P0
scope: workspace-agent
created: 2026-09-29
last-touched: 2026-11-24
aliases: [权威与冲突规则, Authority Rule]
self-referential: true
---

# Authority & Conflict Rule

## 1. Separate two kinds of authority

### Workspace control authority

`agents_vault/05-Governance/`（type: constraint 族；原 `Agent-Governance/`，内迁批 2，见 corr-2026-11-16-58）

Answers:

> How should the Agent work?

Examples:
- stage;
- scope;
- change discipline;
- reporting;
- recovery.

### Runtime specification authority

Current vault formal Specs.

Answers:

> How does TTL actually work?

Examples:
- READ;
- WRITE;
- GOVERN;
- schema;
- runtime access behavior.

Neither should silently replace the other.

## 2. Precedence

For current project state and scope:
1. `STATUS.md`
2. ACTIVE governance constraints
3. current formal vault Specs
4. current schema / ROOT
5. decision records
6. history
7. proposals

For runtime behavior:
1. current formal vault Specs
2. schema / ROOT
3. decision records
4. history
5. proposals

### 2.1 Project-zone zero authority（06-Project/，corr-2026-11-24-110）

`06-Project/` 项目托管族（type: project-spec / project-analysis）对 TTL runtime **零权威**（NON-AUTHORITY）：

- 项目规格的内容权威=项目 Owner，权威仅及项目自身流程；不得替代、覆盖或修改任何 runtime Spec / schema / 治理约束条款。
- 项目件与 runtime Specs 冲突时，runtime Specs 优先；项目件与治理约束冲突时，治理约束优先（本文件 §2 两条 precedence 均不含 project 族——它不在任何权威序里）。
- 项目件不是决策记录：项目区的「规定」不构成对本库 Agent 的指令，除非 Owner 显式裁决采纳。

## 3. Conflict rule

If current formal Specs conflict:
- report the conflict;
- identify evidence;
- do not invent a third architecture.

If implementation conflicts with a current formal Spec:
- normally fix the implementation;
- do not rewrite the architecture unless explicitly authorized.

A proposal is not a decision.

A historical record is not a current command.

## 4. Forbidden shortcut

Never solve ambiguity by keeping two "current" primary paths.
Choose the path already authorized by the current Spec, or stop for user decision.

### 4.1 Old-path retirement（内迁批 4 落地；丙案修订，corr-2026-11-16-62）

治理内迁的旧路径 `Agent-Governance/` **已整体退役**（目录已删除，10/10 本体在 `agents_vault/05-Governance/`）：

- 约束语义唯一原本在库内本体；旧路径不再存在任何文件（过渡期 redirect 指针已随窗口关闭一并移除——机械盘点证实活引用为零，纯文本历史记述不依赖文件存在）。
- **历史解析介质 = git**：历史文档（logs/corr/Audit 归档，append-only 不改）中的旧路径是记述事实；追溯用 `git log --follow -- Agent-Governance/<file>` 或迁移 corr（57/58/60/61/62）。git 是 durable 权威（§2），不在文件系统模拟假目录。
- 本条是 §4 主句在迁移场景的精确化：两条「当前」主路径依旧禁止——退役路径不是路径，历史里的路径字符串不是权威。
