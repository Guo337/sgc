---
id: gov-2026-change
type: constraint
status: ACTIVE
priority: P0
scope: workspace-agent
created: 2026-09-29
last-touched: 2026-11-18
aliases: [变更纪律, Change Discipline]
---

# Minimal Change Discipline

## Standard loop

`READ STATE → ONE TASK → MINIMAL CHANGE → VERIFY → REPORT`

## Before changing anything

State:
- the single task;
- why it belongs to the current stage;
- files expected to change.

## While changing

- change only what the task requires;
- record unrelated findings;
- do not opportunistically refactor unrelated code;
- preserve user work;
- do not delete uncertain history.

## Stop and escalate

Use `NEEDS USER DECISION` when:
- L1/L2/L3 would change;
- authority would change;
- access model would change;
- security boundary would change;
- a new execution entity is required;
- a new knowledge authority is required;
- current Specs are mutually inconsistent;
- the current stage cannot be completed without future-stage work.

## Quotation discipline (引文纪律, corr-2026-11-18-63)

When a later text **revises, supersedes, or executes upon** a load-bearing claim
from an earlier text (design doc, PLAN, card, governance clause):

- embed a Markdown quote block carrying the **original sentence/passage** being
  changed, plus a resolvable pointer (path / wikilink / corr id);
- then write the new statement after the quote block.

Pure narration / addition does NOT trigger this. Paraphrase without revision is
allowed (传意许可, isnad riwaya-bil-ma'na isomorph) but must carry a pointer and
**must not upgrade epistemic modality** (保留→永久保留, 建议→必须 are forbidden
upgrades; see [[表述精确性纪律]] rule 2). Alterations to quoted text must be marked.

Rationale: the quote block forces contact with the original artifact, resetting
each revision chain from paraphrase→paraphrase to original→new. This reduces
compounding drift (drift 2) to single-shot drift (drift 1). Executor = agent
self-discipline (E principle: known-leaky invariant, risk explicitly accepted;
no mechanical gate — T3 deferred, Goodhart risk).

## Forbidden

Do not:
- perform large refactors under the label "cleanup";
- build speculative abstractions;
- replace technology stacks for elegance;
- implement unrelated improvements;
- turn a local fix into an architecture rewrite.
