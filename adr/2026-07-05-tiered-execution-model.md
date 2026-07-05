---
title: Tiered execution model with per-invocation reviewer model routing
date: 2026-07-05
status: Accepted        # Proposed | Accepted | Superseded
supersedes:
superseded-by:
---

# 2026-07-05 — Tiered execution model with per-invocation reviewer model routing

## Context
The template ran one heavyweight pipeline (requirements → plan → implement → review) for
nearly every change, with `opus` on most stages including the orchestrator and the
reviewer. A `/quick-fix` fast-path existed but was chosen manually, was never proposed,
and the orchestrator silently defaulted to the full pipeline. Small and trivial changes
therefore paid for stages and an expensive reviewer they did not need — wasting tokens and
wall-clock time — while the tier a change took was not recorded anywhere. We need
right-sized cost per change without weakening the non-negotiable guardrails (`just check`
on every tier, `protect-primary-checkout.sh`, branch-based work, spec→plan→ADR
traceability for architectural changes).

## Decision
We will define exactly three execution tiers and make tier selection an explicit,
user-confirmed decision:
- **Tier 0 — trivial/hotfix**: branch → implement → `just check` → open PR → one-word
  user confirmation → merge. No code-reviewer stage. All agent work on `sonnet`.
- **Tier 1 — small** (`/quick-fix`): implement → review, spec/plan skipped, code-reviewer
  invoked at `model: sonnet`.
- **Tier 2 — full**: spec → (research) → plan → implement → review, `opus` on
  requirements, planning, and review.
The orchestrator proposes a tier as "Tier N — reply `go`, or name another tier", blocks
on no reply, warns-but-complies on under-tiering, and pauses-and-re-proposes on mid-flight
discovery of architectural scope. There is exactly one `code-reviewer.md`; the
orchestrator supplies its model per-invocation (`sonnet` for Tier 1, `opus` for Tier 2),
which overrides the agent's frontmatter default. The orchestrator stays on `opus`. Every
implementation commit carries a `Change-Tier: trivial|small|full` trailer.

## Alternatives considered
- **Tier 0 merges directly to the default branch with no PR.** Rejected: it would bypass
  the `ci.yml` re-run of `just check` and the "confirm before merging the default branch"
  guardrail, and leave no PR record. Opening a PR is cheap; the savings come from skipping
  spec/plan and the opus reviewer, not from skipping the PR (D4, R2).
- **Two reviewer files (e.g. `code-reviewer.md` + `code-reviewer-lite.md`) to encode the
  cheaper model.** Rejected: duplicated review logic that would drift. A single reviewer
  invoked with a per-invocation `model` override achieves the same routing with no
  duplication (D2, R1).
- **Downgrade the orchestrator off `opus`.** Rejected: tier selection, request
  comprehension, and escalation judgment are high-stakes; the orchestrator stays
  context-lean (paths + short status only), so its `opus` cost is already low (D1).
- **Keep the silent full-pipeline default (status quo).** Rejected: it is the bug — it
  overspends on small changes and hides the tier decision from the user.
- **Auto-merge on green CI.** Rejected: no auto-merge exists and merging stays
  user-confirmed; none is added (R2).

## Consequences
Easier: cost scales to change size; the chosen lane is explicit and auditable
(`Change-Tier` trailer); one reviewer definition to maintain. Harder / trade-offs: the
user must answer a one-word tier prompt each change (block on no reply is intentional);
Tier 0 has no reviewer, so `just check` + CI is the sole automated guard for trivial
changes; routing correctness now depends on the orchestrator passing the right
per-invocation `model`, and on `AGENTS.md`'s documented routing staying in sync with agent
frontmatter (called out in `AGENTS.md`).
