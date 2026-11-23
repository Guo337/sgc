---
id: gov-2026-stage-gate
type: constraint
status: ACTIVE
priority: P0
scope: workspace-agent
created: 2026-09-29
last-touched: 2026-11-16
aliases: [阶段门纪律, Stage Gate]
---

# Stage Gate

## Current stage

`P0 完成`

## Fixed order

`P0 → P1 → P2 → P3 → P4`

## Rules

- A later stage does not become authorized because code for it already exists.
- Current-stage minimum capability comes first.
- Future work goes to backlog.
- Do not package multiple future stages into one implementation.

## P0

Goal:
- doctor-first;
- structural gate;
- current main-path verification;
- deliberate-corruption end-to-end exercise.

Completion:
- P0 = COMPLETE；P0-c deliberate-corruption exercise = PASS（entry_doctor 正确报 BROKEN_LINK → 恢复后 PASS → main 零污染；见 STATUS §5）。

## P1

Goal:
- minimum read orchestration;
- one-hop and targeted two-hop retrieval;
- explainable retrieval;
- baseline measurement.

## P2

Goal:
- reliable session write loop.

## P3

Goal:
- reflection;
- crystallization;
- feedback;
- pitfall;
- rule evolution.

## P4

Goal:
- advanced retrieval such as BM25/RRF/PPR/recency/embedding/Leiden, but only where measured evidence supports it.

## Stop condition

If completing the current task requires architecture outside the current stage:
- report;
- do not silently expand.
