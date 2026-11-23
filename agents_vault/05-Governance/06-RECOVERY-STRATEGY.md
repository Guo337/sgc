---
id: gov-2026-recovery
type: constraint
status: ACTIVE
priority: P0
scope: workspace-agent
created: 2026-09-29
last-touched: 2026-11-16
aliases: [收线策略, Recovery Strategy]
---

# Recovery / 收线策略

## 1. When to enter Recovery

Enter Recovery when:
- multiple implementations compete;
- current and historical rules are mixed;
- Specs reference missing "ghost" implementations;
- later-stage work exists before the current stage is accepted;
- experiments are mixed with production paths;
- every task expands into several more tasks.

Goal:

> **Shorten the front, restore a single main path, establish a verifiable baseline.**

## 2. Recovery priority

`FACT INVENTORY > MAIN-PATH CONVERGENCE > VERIFICATION > DOCUMENT SYNC > NEW FEATURES`

## 3. Three buckets

### CURRENT
Current, authorized, actually used.

### EXPERIMENTAL
Existing but not in the current main path.

### ARCHIVE
Historical, superseded, duplicated, or retired.

Do not delete uncertain assets during the first pass.

## 4. Inventory first

Do not refactor while discovering.

Record:

| Asset | Type | Actual use | State | Spec basis | Callers |
|---|---|---|---|---|---|

Pay special attention to:
- duplicate scripts;
- duplicate entry points;
- duplicate retrieval logic;
- multiple access paths;
- unreferenced files;
- ghost implementations;
- future-stage work.

## 5. Establish the shortest main path

Current workspace control path:

`AGENTS.md → agents_vault/05-Governance/00-ENTRY.md → STATUS.md → current formal Spec → current implementation → doctor/self-test`

For current P0 work:

`state → Spec → P0 implementation → doctor → self-test`

Do not require an Agent to traverse the entire project for a P0 task.

## 6. One capability = one CURRENT

For a responsibility such as "read":

- one CURRENT primary;
- explicit FALLBACK paths;
- experimental alternatives remain experimental;
- old paths become LEGACY/ARCHIVE.

Do not keep two primary paths merely because both work.

## 7. Preserve history

Historical experiments are valuable as evidence.

Prefer:

`identify → isolate → verify → archive`

over immediate deletion.

## 8. Freeze future lines

During current P0/P1 stabilization, do not independently implement:
- embedding;
- Leiden;
- advanced PPR/RRF;
- large-scale community compilation;
- new database/vector store;
- second knowledge authority;
- new execution entity;
- speculative extensibility layers.

Record these as backlog/experimental work.

## 9. Cleanup is not redesign

Allowed:
- consolidate duplicate entry points;
- move clearly experimental material out of the main path;
- repair P0 blockers;
- add focused tests;
- update current status;
- remove clearly dead code once reversible.

Not allowed:
- architecture rewrite;
- technology replacement;
- new major abstraction;
- unrelated optimization.

## 10. Task exit condition

Every Recovery task must end with a concrete answer:

> **What became more singular, more understandable, or more verifiable?**

If nothing did, the task probably expanded incorrectly.

## 11. Recovery completion

Recovery is complete only when:
- a new Agent can identify the current stage quickly;
- each core capability has one CURRENT path;
- CURRENT paths are verifiable;
- experiments/history are isolated;
- Git can recover previous states;
- future work is not mixed into the current path.
