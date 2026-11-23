---
id: gov-2026-terms
type: reference
status: ACTIVE
priority: P2
scope: workspace-agent
created: 2026-11-07
last-touched: 2026-11-16
aliases: [术语对照, Canonical Tokens]
---

# Canonical Tokens (EN) / 术语对照

## 0. What this file is

A **naming index**, not a semantic authority.

- It records the English tokens this workspace already uses, with a pointer to where each one is defined.
- It does not define the TTL runtime, the L1/L2/L3 model, or any audit semantics.
- If a gloss below disagrees with its cited source, **the cited source wins** and the disagreement must be reported as `CONFLICT`.

## 1. The bilingual rule

> 中文承载语义，英文承载标识。

- Prose stays Chinese. Operative meaning is carried by Chinese prose.
- States, verdicts, and field names are fixed English tokens. They are the machine-facing identifier and must be written exactly as listed — no translation, no case change, no local synonym.
- Do **not** invent an English token for a concept that has no established one. If a concept needs a token, that is a `NEEDS USER DECISION`, not a glossary edit.
- If Chinese prose and an English token in the same document point at different states, that is a `CONFLICT` to report — never silently pick one.

Why: the failure mode being defended against is not "difficulty reading English". It is **两个不同的词被当成同一个状态**, e.g. `PROVISIONAL` read as `PASS`, or `MINOR DIVERGENCE` collapsed into `MATCH`.

## 2. Governance frontmatter fields

| Field | Fixed values | Meaning |
|---|---|---|
| `id` | `gov-2026-<topic>` \| `gov-status`（status 单例专名） | Card identifier. Stable; never reused. |
| `type` | `constraint` \| `reference` \| `status` | `constraint` = binding (must be read per `00-ENTRY.md` §3). `reference` = consultable, not binding. `status` = 当前态指针单例（批3，schema governance 节）。 |
| `status` | `ACTIVE` \| `SUPERSEDED`（须 `supersedes` 指针；type:status 单例仅 `ACTIVE`） | In force. Cards are removed or superseded, not silently left stale. |
| `priority` | `P0` \| `P1` \| `P2` | `P0` = violation blocks the task.（constraint 必填；status/reference 不要求） |
| `scope` | `workspace-agent` | Applies to Agent behavior in this workspace only. |

## 3. Report block headings

`04-REPORTING.md` defines the template; `00-ENTRY.md` §7 lists the operative subset.

`Completed` / `Changed` / `Why` / `Verified` / `Not Changed` / `Remaining` / `Conflicts / Decisions` / `Current Stage` / `Next Single Task`

- `Completed` — what the task set out to do and whether it was achieved.
- `Changed` — files actually modified, with paths.
- `Why` — the reason the change belongs to the current stage.
- `Verified` — the command that was run and its observed result. `Verified` means **observed**, not "should work".
- `Not Changed` — things a reader might expect to move but deliberately did not.
- `Remaining` — known incomplete items, including ones deliberately deferred.
- `Conflicts / Decisions` — `CONFLICT` or `NEEDS USER DECISION` blocks raised this round.
- `Current Stage` — the stage string from `STATUS.md`.
- `Next Single Task` — exactly one next action, not a list.

## 4. Evidence classes

| Token | Meaning |
|---|---|
| `FACT` | Observed on disk, by test, or by documented current authority. |
| `INFERENCE` | A reasoned interpretation of facts. |
| `PROPOSAL` | A suggested design, not in force. |
| `USER DECISION` | A choice that only the user can make. |

Rule: a `PROPOSAL` must never be written into a document as if it were a current rule.

## 5. Escalation and conflict blocks

`NEEDS USER DECISION` — the Agent stops. Fields: `Question` / `Evidence` / `Why it matters` / `Options`.

`CONFLICT` — two authorities or two records disagree. Fields: `Topic` / `Evidence A` / `Evidence B` / `Current authority` / `Impact` / `Action`.

Distinction that matters: `CONFLICT` is *evidence has been gathered and the sources disagree*. `NEEDS USER DECISION` is *the Agent cannot proceed without a ruling*. A conflict usually produces a decision, but a decision does not require a conflict.

## 6. Stage, seal, and certification states

| Token | Meaning |
|---|---|
| `SEALED` | Frozen artifact set. Content is not edited; corrections require a new version. |
| `IMMUTABLE` | Same force as `SEALED`; used when stating that an artifact may not be rewritten at all. Integrity is pinned by blob sha256 in a manifest. |
| `PROVISIONAL` | Executed and recorded, but **not** certified. Readable as evidence; not usable as a certified result. |
| `UN-CERTIFIABLE` | The evidence as designed cannot support certification, regardless of how the run went. Distinct from `FAIL`: nothing was proven either way. |
| `certified PASS` | The only phrase that asserts a criterion was met under certification. `PASS` alone is not this. |
| `MECHANICAL PASS` | The mechanical checks passed. Says nothing about semantic adequacy. |
| `SEMANTIC PARTIAL` | Some semantic checks are unmet or unevaluable. |
| `deferred` | Deliberately not executed this round. Not a failure, not a pass. |
| `frozen` / `frozen_for_retry` | Manifest states. `frozen_for_retry` records that a retry is planned against the same frozen baseline. |

Forbidden promotion: `PROVISIONAL` → `certified PASS`. A provisional result may only be replaced by a new executed round, never by relabeling.

## 7. Audit verdicts

| Token | Meaning |
|---|---|
| `MATCH` | The artifact agrees with the compared intent/record. |
| `MINOR DIVERGENCE` | Disagrees in wording, scope framing, or granularity; does not change the substantive judgment. |
| `MAJOR DIVERGENCE` | Disagrees substantively. Requires a cause: sample-design ambiguity, erroneous intent, or Owner review. |

Rule: never adjust the artifact to reach `MATCH`. Divergence is recorded first, cause assigned second.

## 8. Gold relation set names

Observed set names, exact spelling:

`GOLD.true_links` / `GOLD.false_links` / `GOLD.rejected_links` / `GOLD.ambiguities` / `GOLD.insufficient` / `GOLD.conflicts` / `GOLD.required_conflicts` / `GOLD.required_uncertainties` / `GOLD.required_corrections` / `GOLD.forbidden_promotions` / `GOLD.provenance_requirements` / `GOLD.initial_claims` / `GOLD.confirmations` / `GOLD.noise`

Which names are populated, and what counts as membership, is defined per round in that round's Gold document. This table only fixes spelling.

## 9. Registration code families

| Pattern | Meaning |
|---|---|
| `G-<n>` | Contract gap. Opened when a contract cannot decide a case; closed only by a ruling. |
| `P<n>` | Probe / fixture defect. Distinct namespace from the stage name `P1`. |
| `F-<n>` | Finding. |
| `D<n>` | Decision item within a round's decision register. |
| `B<n>` / `B1-02` | Round-scoped sub-item identifiers. |
| `C<n>` / `T<n>` | Criterion / test identifiers, defined in that round's protocol. |

Ambiguity to watch: `P1` is both a **stage name** (`P1 active`) and a **probe-defect number** (`P7–P8`). Disambiguate by context word, and prefer writing the stage as `P1 stage` when proximity to probe defects makes it unclear.

## 10. Known divergences

- `00-ENTRY.md` §7 lists eight report blocks; `04-REPORTING.md` lists nine (adds `Completed`). The template is a superset. Not resolved here — recorded so a reader does not treat the shorter list as the full one.

## 11. Maintenance

- Adding a token: append with a pointer to its defining source.
- Retiring a token: mark it `SUPERSEDED`, do not delete; older documents still use it.
- Never redefine an existing token to match new practice. If practice changed, the old token keeps its old meaning and the new practice gets a new token.
