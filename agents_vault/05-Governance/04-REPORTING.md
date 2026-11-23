---
id: gov-2026-reporting
type: constraint
status: ACTIVE
priority: P0
scope: workspace-agent
created: 2026-09-29
last-touched: 2026-11-18
aliases: [汇报纪律, Reporting Discipline]
---

# Report Facts, Not Guesses

## Principle

> Fact → evidence → action.

Distinguish:

- `FACT` — observed on disk, by test, or by documented current authority.
- `INFERENCE` — a reasoned interpretation.
- `PROPOSAL` — a suggested design.
- `USER DECISION` — a choice requiring the user.

Never present a proposal as a current rule.

## Epistemic-strength tags in design texts (corr-2026-11-18-63)

Strong claims in design documents / PLANs / cards (永久 / 必须 / 绝不 / 唯一 /
"by design" / "permanent") must carry one of:

- `[VERIFIED: <evidence>]` — checked against the artifact (disk scan, test,
  mechanical census) at the time of writing;
- `[ASSUMED: <pending verification>]` — a planning-time judgment not yet
  verified; **must be re-verified against the artifact when executed upon**
  (planning-time judgment does not exempt execution-time falsification).

This extends the FACT/INFERENCE convention above from reports to durable design
texts. Untagged strong claims in legacy texts are treated as ASSUMED.
Executor = agent self-discipline (known-leaky, risk accepted; see E principle).

## Required task report

```text
## Completed
...

## Changed
...

## Why
...

## Verified
...

## Not Changed
...

## Remaining
...

## Conflicts / Decisions
...

## Current Stage
...

## Next Single Task
...
```

## Conflict format

```text
CONFLICT
- Topic:
- Evidence A:
- Evidence B:
- Current authority:
- Impact:
- Action:
```

## Escalation format

```text
NEEDS USER DECISION
- Question:
- Evidence:
- Why it matters:
- Options:
```

Do not continue around a genuine architectural blocker by pretending it is only an implementation issue.
