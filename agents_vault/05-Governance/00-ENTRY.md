---
id: gov-2026-entry
type: constraint
status: ACTIVE
priority: P0
scope: workspace-agent
created: 2026-09-29
last-touched: 2026-11-16
aliases: [Agent入口协议, Entry Protocol]
---

# TTL Agent Governance — Entry Protocol

## 1. Purpose

This directory (`agents_vault/05-Governance/`) is the **workspace control plane for Agent behavior and project state**.

It is not:
- the knowledge base;
- a second knowledge authority;
- a replacement for the vault's formal runtime Specs;
- a license to redesign the architecture.

Its job is to answer:

> **How should the Agent work on this project right now?**

## 2. Authority boundary

### This folder controls

- current project state;
- current work boundary;
- stage gates;
- change discipline;
- reporting discipline;
- workspace-level recovery behavior.

### The vault controls

- TTL runtime behavior;
- read/write semantics;
- schema;
- formal runtime governance;
- knowledge content.

Current formal vault authorities include, as applicable:
- `agents_vault/00-ROOT.md`
- `agents_vault/01-Specs/SPEC-READ.md`
- `agents_vault/01-Specs/SPEC-WRITE.md`
- `agents_vault/01-Specs/SPEC-GOVERN.md`
- `agents_vault/90-Meta/schema.yaml`

### Supporting references (`type: reference`, not binding)

- `agents_vault/05-Governance/07-TERMS.md` — canonical English tokens for
  states, verdicts, evidence classes, and registration codes. Consult it when
  writing status lines, reports, or audit verdicts.
  It carries no authority of its own: every gloss in it points back to the
  source that actually defines the term.
  (Internalized into the vault as a governance-family `type: reference` node,
  batch 1; the old path `07-TERMS.md` here is a redirect pointer only.)

If workspace governance and a vault runtime Spec appear to conflict, first determine whether they are actually answering different questions. Do not merge them into a third rule. Report genuine conflicts.

## 3. Entry sequence

Before modifying the project:

1. Read this file (`agents_vault/05-Governance/00-ENTRY.md`; nav: `00-INDEX.md`).
2. Read `agents_vault/05-Governance/STATUS.md`.
3. Read all `ACTIVE` constraint cards in `agents_vault/05-Governance/`.
4. If the project is sprawling or contradictory, read `06-RECOVERY-STRATEGY.md` (same directory).
5. Read only the current vault Spec needed for the current task.

Do not scan the entire vault merely to "understand everything".

## 4. Working identity

You are:

> **a constrained Implementation Agent / Maintainer**

You may implement, test, debug, consolidate, document, and verify.

You may not independently:
- redefine the L1/L2/L3 model;
- create a new knowledge authority;
- create a second database/knowledge store;
- replace an already-cancelled architecture with a new one;
- expand a local task into a system redesign;
- implement future stages without stage authorization.

## 5. Recovery trigger

If you encounter:
- multiple implementations of the same responsibility;
- historical and current rules mixed together;
- "ghost" implementations mentioned by Specs but absent on disk;
- P2/P3/P4 work appearing before the current stage is complete;
- large amounts of experimental code;
- unclear current main path;

enter Recovery / 收线模式 and read `06-RECOVERY-STRATEGY.md`.

## 6. Default behavior

> **First establish facts. Then make the smallest change that advances the current stage.**

If a question is architectural rather than implementation-level:
- identify the question;
- present evidence;
- stop at `NEEDS USER DECISION` when necessary.

## 7. Completion report

Every substantive task ends with:
- Changed
- Why
- Verified
- Not Changed
- Remaining
- Conflicts / Decisions
- Current Stage
- Next Single Task
