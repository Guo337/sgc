---
id: gov-2026-blind-isolation
type: constraint
status: ACTIVE
priority: P1
scope: workspace-agent
created: 2026-11-13
last-touched: 2026-11-16
aliases: [盲态执行隔离, Blind Execution Isolation]
---

# Blind Execution Isolation

Owner adjudications:
- 2026-11-13 (a): the Round 3C `session_context` leak (D6) is accepted; the five
  gates cross-validate.
- 2026-11-13 (b): **calibrated down**. Blind isolation is an experiment fixture,
  not a product property. Stop treating a disclosed *pointer* as contamination.

This card governs **how the Agent dispatches blind sub-agents**. It creates no
runtime authority and changes no vault Spec.

## 1. Scope — and where it deliberately does not apply

Applies only when a **measurement** depends on the executor not seeing prior
answers: blind Gold writing, blind Runs (A/B/C), independent reviews.

**Does not apply to ordinary work, and must never be imported into runtime
behavior.** SPEC-READ exists to *assemble* relevant history for the Agent
(ROOT routing, budgets, recency, backlinks). Pursuing blindness in real use would
disable the system's core function. Blindness is the clamp used while taking a
measurement, removed afterwards.

## 2. What blindness actually protects

Not "the AI might cheat". It protects **what H2/H4 measure**:

- H2 asks whether two mutually unaware executors converge on the same terminal
  state from the same input. If B can see A's answer, the number degrades into a
  copying rate.
- H4 asks the same across time. If Run C knows A2's result, agreement is trivial.
- Gold must be an independent baseline; writing it while reading DATASET §3
  construction intent would make S1 self-confirming.

## 3. Threat model — reachability, not disclosure

A leak matters only if the executor can **act** on it. Two distinct things:

| | Meaning | Weight |
|---|---|---|
| **Disclosure** | a title/filename hints that a round, runs, or defects exist | **low** — cannot be turned off, and derives no cell's verdict |
| **Reachability** | the executor can actually read answer-bearing content | **decisive** — this is what invalidates a run |

Verified facts (Round 3C):
- V1: the harness auto-injects `<session_context>` listing checkpoint filenames
  and titles into every sub-agent system prompt. Not suppressible via prompt.
- V2: those titles may name the round and its runs. They contain **no sample
  bodies, no Gold terminal states, no workspace path**.
- V3: the named files on disk may contain answers — so *reading* one would be
  real contamination.
- V4: reusing an agent context destroys independence outright.
- V5: an executor may self-exempt from tool discipline (Run B called `sql` twice).

**With zero tool calls, V1–V3 are an unloaded gun**: the pointer exists but no
read is physically possible. That is the real safeguard, and it is cheap.

## 4. Rules

- **R1 — Fresh agent, always.** One dispatch = one new agent. Never `write_agent`
  into a blind executor, never resume. Verify afterwards: distinct agentId,
  `resumable: false`, exactly one user message.
- **R2 — Zero tool calls.** The prompt must forbid all tool calls, including
  session-state tools (`sql`, memory, file reads). This is the load-bearing rule:
  it removes reachability rather than arguing about disclosure.
- **R3 — Post-flight verification.** After the run, check the event log for tool
  calls and check the reasoning trace for references to session artifacts.
  Archive the result. **A tool call, or an actual reference to answer-bearing
  content, invalidates the run.**
- **R4 — Disclose, do not overclaim.** Never write "prompt is the entire input"
  or "hard isolation" unqualified. The accurate claim is **instructional
  isolation plus verified zero reachability**. Firewall attestation stays
  Owner-only.
- **R5 — Pre-flight is informational.** Run `isolation_preflight.py` and archive
  its report so the disclosure is on record. It **does not block dispatch**.
- **R6 — Escalate only on reachability.** Stop and report `NEEDS USER DECISION`
  only if the executor could actually reach answer-bearing content (e.g. tools
  are required by the task, or a tool call occurred). A disclosed title alone is
  not grounds to stop.

## 5. Calibration record (why R7 was removed)

The first version of this card had **R7: stop on any material disclosure**, and
its pre-flight exited 1 on disclosed titles. That blocked Run C and E5 over
checkpoint *names*, while the runs in question had **zero tool calls** and could
not read anything.

That was over-defensive: it treated an unreachable pointer as contamination and
would have pushed the Agent to edit harness session files — damaging the Owner's
checkpoint history to fix a non-problem. Owner flagged the pattern directly
("过度偏执"). The card now weights **reachability**, matching
`operating-mode`'s principle that over-guarding a zero-risk reversible action is
itself an error.

Retained without weakening: fresh agents, zero tool calls, post-flight
verification, honest disclosure. **Nothing about H1–H5 or any frozen instrument
changed.**

## 6. Tooling

`Audit/Current/TTL-Core-Hypothesis-Spec-NON-AUTHORITY/ROUND-3C-RAW/tools/`

- `isolation_preflight.py` — informational disclosure report (R5), always exit 0.
- `verify_isolation.py` — post-dispatch verification (R1/R2/R3).

Both read-only with respect to the vault and all frozen instruments.