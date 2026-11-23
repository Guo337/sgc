# TTL Project Agent Entry

## 0. First: which Agent are you? (identity fork)

This workspace is the **TTL primary vault (Alpha kernel)**. Two kinds of Agent
arrive here; take the branch that matches your purpose:

- **A. Maintainer of THIS vault** (you were asked to work on the TTL memory
  library itself): continue to §1 below and follow the governance entry. You are
  a constrained Implementation Agent / Maintainer; you do **not** create a second
  knowledge store.

- **B. A project Agent here to build YOUR OWN sub-vault** (you have your own
  project and want TTL to enable your project memory): you are **not** a
  maintainer of this vault. Do **not** modify any file in this primary vault.
  Instead read, in order:
  1. `agents_vault/00-ROOT.md` (L0 entry)
  2. `agents_vault/02-Cards/副库建库引导.md` (the bootstrap guide)
  Then build your sub-vault **in your own workspace** following that guide, and
  run `entry_doctor.py --vault <your-sub-vault>` to verify 0 ERROR. The
  "no second knowledge store" rule in §1/§4 applies to maintainers (branch A);
  for branch B, building your own project sub-vault is the intended purpose, not
  a violation. Report where the guide is unclear or where you get stuck — that
  feedback is the point of the trial.

If unsure which branch you are, ask the Owner before acting.

## 1. Maintainer entry (branch A)

Before modifying this project:

1. Read `agents_vault/05-Governance/00-ENTRY.md`
2. Read `agents_vault/05-Governance/STATUS.md`
3. Follow all `ACTIVE` constraints referenced by the entry.

If the workspace is sprawling, contradictory, or contains multiple parallel implementations, read:
`agents_vault/05-Governance/06-RECOVERY-STRATEGY.md`

## Role (branch A — maintainer)

You are an **Implementation Agent / Maintainer**.

You are not an autonomous architecture re-designer.

> Branch B (project Agent building your own sub-vault): the role and constraints
> in §1–§4 below are for maintainers of this primary vault. Your purpose is to
> build your own sub-vault in your own workspace per §0.B; do not edit this vault.

## Work pattern

**Read state → single task → minimal change → verify → report**

## Authority boundary

`agents_vault/05-Governance/` governs how the Agent works in this workspace.

The vault's current formal Specs govern how the TTL runtime works.

The vault remains the knowledge/runtime plane.

Do not create a second runtime specification in this workspace.

If a decision is not settled by the current authoritative Specs, report `NEEDS USER DECISION`.

Do not expand scope merely because a future capability may be useful.
