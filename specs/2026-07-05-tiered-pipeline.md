---
title: Tiered execution pipeline with model-routing cost reductions
date: 2026-07-05
status: Delivered        # Draft | Ratified | Delivered
---

# 2026-07-05 — Tiered execution pipeline with model-routing cost reductions

## Objective
The template runs one heavyweight pipeline (requirements → plan → implement → review) for
nearly every change, and almost every stage — including the orchestrator and reviewer —
runs on `opus`. A `/quick-fix` fast-path exists but is chosen manually, is never proposed,
and the orchestrator silently defaults to the full pipeline; even the fast-path and the
orchestrator itself are expensive. This wastes tokens and wall-clock time on small changes.
We want an **explicit three-tier execution model** where the orchestrator *proposes* a tier
(based on the trivial-change boundary already defined in AGENTS.md) and the user confirms or
overrides in one word, plus **model-routing cost reductions** (tiered reviewer model via a
per-invocation override). Non-negotiable safety guardrails — the `just check` gate on every
tier, the `protect-primary-checkout.sh` hook, branch-based work, and spec→plan→ADR
traceability for architectural changes — must be preserved. The value: right-sized cost per
change without weakening quality controls, and a git history that records which lane each
change took.

This is a documentation/configuration change to the template itself: the deliverables are
Markdown agent definitions (`.claude/agents/*.md`), slash commands (`.claude/commands/*.md`),
a new `/hotfix` command, `AGENTS.md`, and the relevant READMEs. Acceptance criteria are
therefore verifiable by inspecting the resulting files. No shell hooks are modified.

## User stories
- As a template user making a one-line fix, I want the orchestrator to propose a cheap
  "trivial" lane so I don't pay for a spec, a plan, and an opus reviewer I don't need.
- As a template user, I want to see which tier the orchestrator chose and be able to
  override it in one word, so the tier is an explicit, auditable decision rather than a
  silent default.
- As a template maintainer, I want each change's commit history to record the tier it took
  (`Change-Tier: trivial|small|full`) so I can audit whether the right lane was used.
- As a template owner watching cost, I want the lower-tier reviewer to run on a cheaper
  model, without losing opus rigor on genuinely architectural (Tier 2) changes.
- As a template user, I want the safety guardrails (quality gate on every tier, no writes to
  the primary checkout on the default branch, branch-based work) preserved no matter which
  tier runs, so speed never costs correctness or repo safety.

## Functional requirements

### Tier model
1. The workflow MUST define exactly three tiers with these flows:
   - **Tier 0 — Trivial / hotfix**: a localized fix with no new dependency, interface, or
     schema change, and nothing ADR-worthy. Flow: branch → implement → `just check` →
     open PR → user confirmation → merge. There is **no code-reviewer stage**. All agent
     work uses `sonnet`.
   - **Tier 1 — Small change** (today's `/quick-fix`): a scoped change with no schema or
     interface change and nothing ADR-worthy beyond routine. Flow: implement → review
     (spec and plan skipped). The **code-reviewer runs on `sonnet`** (downgraded from opus).
     Behaviorally identical to today's `/quick-fix` in every respect except the reviewer
     model (D8).
   - **Tier 2 — Full pipeline**: any change introducing a new dependency, interface, or
     schema change, or that is otherwise ADR-worthy. Flow: spec → (research as needed) →
     plan → implement → review, with `opus` on the high-stakes stages including the reviewer.
2. Tier assignment MUST be driven by the "trivial change" boundary already stated in
   AGENTS.md (no new dependency/interface/schema change, nothing ADR-worthy). The three-tier
   text in AGENTS.md and in the agent/command files MUST be mutually consistent (single
   source of truth for the boundary, referenced elsewhere rather than re-stated divergently).

### Orchestrator propose-and-confirm behavior
3. On receiving a change request, the orchestrator MUST **propose** a tier (0, 1, or 2)
   with a one-line rationale tied to the boundary, and MUST present it to the user for
   confirmation or override before starting that tier's flow. It MUST NOT silently default
   to the full pipeline.
4. The orchestrator MUST propose the tier in the form "Tier N — reply `go`, or name another
   tier", so the user accepts with a one-word `go` or overrides by naming a different tier
   (D5). If the user does not reply, the orchestrator MUST block and MUST NOT proceed on any
   default (D5).
5. When the orchestrator proceeds, it MUST record the selected tier and pass it to every
   downstream stage so the tier is available for the commit trailer and for model routing.

### Model routing
6. The **orchestrator** agent MUST remain on `opus` (D1): tier selection, request
   comprehension, and escalation judgment are high-stakes, and the orchestrator stays
   context-lean (it passes only file paths and short status between stages, never artifact
   content) so its opus cost is low. Its documentation SHOULD note non-behaviorally that it
   does not require a 1M-context window.
7. There MUST be exactly **one** `code-reviewer.md` agent definition. The orchestrator MUST
   invoke it with a per-invocation `model` override — `model: sonnet` for Tier 1 reviews and
   `model: opus` for Tier 2 reviews (D2, enabled by R1: the Agent/Task tool's per-invocation
   `model` takes precedence over the agent's frontmatter `model:`). Tier 0's flow has no
   reviewer stage.
8. The `Model routing` section of AGENTS.md MUST be updated so the documented routing matches
   the actual `model:` frontmatter of every agent after this change, and MUST document that
   the code-reviewer's effective model is set per-invocation by the orchestrator (sonnet for
   Tier 1, opus for Tier 2) rather than fixed by its frontmatter (no drift between the doc
   and the files).
9. Model routing for the other stages MUST be unchanged: researcher `haiku`;
   implementation-engineer `sonnet`; requirements-engineer and technical-planner `opus`
   (these run only in Tier 2); orchestrator `opus`.

### Traceability trailer
10. Every implementation commit produced by the pipeline MUST carry a git trailer recording
    the tier, in the form `Change-Tier: trivial | small | full` (lowercase; one value per
    commit, matching the tier that ran). The **implementation-engineer** MUST add this
    trailer on the **final commit** it produces (D9).
11. The spec → plan → ADR chain MUST remain unchanged and MUST continue to apply only to
    Tier 2. Tier 0 and Tier 1 produce no spec/plan; Tier 0 additionally produces no ADR
    (by definition nothing ADR-worthy reaches Tier 0).

### Guardrails (constraints — non-negotiable)
12. `just check` (the full quality gate) MUST run and pass before merge on **every** tier,
    including Tier 0.
13. The `protect-primary-checkout.sh` hook (and all other existing hooks) MUST NOT be
    disabled, weakened, or modified. All tiers, including Tier 0, MUST perform their edits on
    a feature branch in a worktree, never as direct writes to the primary checkout while it
    is on the default branch.
14. Tier 0 MUST **open a PR** after `just check` passes (D4) — this is cheap, lets the
    existing `.github/workflows/ci.yml` re-run `just check` on the PR, and keeps a record.
    Merging MUST require a **one-word user confirmation** before the default branch is merged
    (D3), preserving the AGENTS.md "confirm with the user before merging the default branch"
    rule. There is no auto-merge, and none is added (R2).
15. The merge gate MUST remain, per tier: Tier 2 and Tier 1 merge only on
    `reviewer-clean AND local-gate-clean`; Tier 0 (no reviewer) merges on
    `local-gate-clean AND one-word-user-confirmation`.

### Backward compatibility & command surface
16. The existing `/spec`, `/plan`, `/implement`, and `/review` slash commands MUST continue
    to work and MUST map to the Tier 2 flow. The existing `/quick-fix` command MUST continue
    to work and MUST map to the Tier 1 flow (its reviewer downgraded to sonnet per FR7).
17. A new **`/hotfix`** slash command MUST be added for Tier 0, mirroring the shape and
    conventions of `/quick-fix` (D6). Tier 0 is reachable both via `/hotfix` and via
    orchestrator auto-proposal.

## Edge cases
- **Mid-flight escalation.** During Tier 0 or Tier 1 implementation, the change is
  discovered to need a new dependency/interface/schema change or an ADR. The orchestrator
  MUST **pause and re-propose a higher tier** to the user rather than smuggle an
  architectural change through a lower tier (D7).
- **User overrides below the boundary.** The user picks a tier below what the boundary
  warrants (e.g., Tier 0 for a schema change). The orchestrator MUST **warn but comply**
  (D7) — it states the mismatch, then runs the user-selected tier.
- **User gives no tier reply.** The orchestrator blocks and awaits input; it MUST NOT
  proceed on any default (that is the current bug) (D5).
- **Tier 0 gate failure.** If `just check` fails on Tier 0, the change MUST NOT merge; the
  fix continues on the branch until the gate is green (no reviewer exists to catch it, so the
  gate is the sole automated guard).
- **protect-primary-checkout default-branch detection fails.** The hook fails safe (allows
  the write) today; Tier 0 MUST NOT rely on the hook as its only protection against writing
  to the default branch — it MUST work on a branch by construction.
- **Ambiguous request at proposal time.** If the orchestrator cannot classify the change,
  it MUST ask rather than guess a tier.

## Acceptance criteria
Given/When/Then, verifiable by inspecting the resulting repository files.

- **Given** the orchestrator definition after this change, **when** `.claude/agents/orchestrator.md`
  is read, **then** it instructs the orchestrator to propose one of three named tiers with a
  boundary-based rationale in the form "Tier N — reply `go`, or name another tier", to obtain
  user confirmation/override before running a tier's flow, and it no longer permits a silent
  default to the full pipeline; and it states that on no reply the orchestrator blocks and
  does not proceed.
- **Given** the orchestrator definition, **when** its `model:` frontmatter is read, **then**
  it is `opus` (unchanged), and the file notes it does not require a 1M-context window.
- **Given** AGENTS.md after this change, **when** its "Scale to the task" section is read,
  **then** it describes the three tiers (Tier 0 trivial/hotfix, Tier 1 small, Tier 2 full)
  with their flows and the boundary that selects each.
- **Given** AGENTS.md, **when** its "Model routing" section is read, **then** the documented
  model for every agent matches that agent file's `model:` frontmatter with no discrepancy,
  and it states that the code-reviewer's effective model is set per-invocation by the
  orchestrator (sonnet for Tier 1, opus for Tier 2).
- **Given** the repository after this change, **when** `.claude/agents/` is listed, **then**
  there is exactly one code-reviewer definition (`code-reviewer.md`) and no second/`-lite`
  reviewer variant.
- **Given** the orchestrator definition, **when** its Tier 1 and Tier 2 review steps are
  read, **then** each instructs invoking `code-reviewer.md` with a per-invocation `model`
  override — `model: sonnet` for Tier 1 and `model: opus` for Tier 2.
- **Given** the Tier 0 flow definition (orchestrator and `/hotfix`), **when** it is read,
  **then** it contains branch → implement → `just check` → open PR → one-word user
  confirmation → merge, with **no code-reviewer stage**, and states that all Tier 0 agent
  work uses `sonnet`.
- **Given** the Tier 0 flow definition, **when** it is read, **then** it requires `just check`
  to pass before merge, requires work on a feature branch/worktree (never direct writes to
  the primary checkout on the default branch), and requires a one-word user confirmation
  before the default branch is merged.
- **Given** the Tier 1 flow definition (updated `/quick-fix`), **when** it is read, **then**
  it runs implement → review with the reviewer invoked at `model: sonnet`, and is otherwise
  identical to the current `/quick-fix` flow.
- **Given** the implementation-engineer definition after this change, **when** its commit
  guidance is read, **then** it requires a lowercase `Change-Tier: trivial|small|full`
  trailer on the final commit it produces, matching the tier that ran.
- **Given** the `.claude/hooks/` directory, **when** each hook (including
  `protect-primary-checkout.sh`) is diffed against its pre-change version, **then** it is
  unchanged — no hook is disabled, weakened, or modified.
- **Given** the Tier 2 flow definition, **when** it is read, **then** it retains
  spec → (research) → plan → implement → review with `opus` on requirements, planning, and
  review, and the spec→plan→ADR chain intact.
- **Given** the slash commands after this change, **when** each is read, **then**
  `/quick-fix` resolves to Tier 1; `/spec`, `/plan`, `/implement`, `/review` resolve to the
  Tier 2 flow; and a new `/hotfix` command exists that resolves to the Tier 0 flow and
  mirrors `/quick-fix`'s shape and conventions.
- **Given** the workflow definitions, **when** the escalation guidance is read, **then**
  there is an explicit instruction that (a) a Tier 0/1 change discovered mid-flight to be
  architectural causes the orchestrator to pause and re-propose a higher tier, and (b) an
  under-tier user override is met with a warning but complied with (D7).

## Non-functional requirements
- **Cost/efficiency (primary driver).** The change MUST reduce expected token/compute cost
  for trivial and small changes versus today (fewer stages on Tier 0; sonnet reviewer on
  Tier 1). This is verified by inspection of the routing/flow files, not by a runtime
  benchmark (documentation/config repo).
- **Consistency.** No drift between AGENTS.md's stated routing/boundary and the actual agent
  frontmatter and command flows — a single boundary definition referenced from all sites.
- **Safety preservation.** No guardrail (quality gate, primary-checkout protection,
  branch-based work, secret scan) is weakened by any tier.
- **Auditability.** The tier of each change is recoverable from git history via the trailer.
- **Clarity for the human.** The tier proposal and override interaction must be simple enough
  to answer in one word.
- **Style.** All edited Markdown files follow existing template conventions (tone,
  structure, frontmatter shape) per AGENTS.md.

## Assumptions & decisions (all resolved)
All prior open questions are resolved; each is recorded below as an accepted decision or
assumption. No open questions remain.

- `[DECISION | D1 — accepted-by-user]` The orchestrator stays on `opus` (unchanged). Tier
  selection, request comprehension, and escalation judgment are high-stakes, and the
  orchestrator stays context-lean so its opus cost is low. Documented note: it does not need
  a 1M-context window.
- `[DECISION | D2 — accepted-by-user]` One `code-reviewer.md`; the orchestrator invokes it
  with `model: sonnet` for Tier 1 reviews and `model: opus` for Tier 2 reviews. No duplicate
  reviewer files.
- `[DECISION | D3 — accepted-by-user]` Tier 0 merge requires a one-word user confirmation
  before the default branch is merged, preserving the AGENTS.md guardrail. No code-reviewer
  stage for Tier 0.
- `[DECISION | D4 — accepted-by-user]` Tier 0 still opens a PR (cheap; lets `ci.yml` run
  `just check`, keeps a record). Savings come from skipping spec/plan and the opus reviewer,
  not from skipping the PR.
- `[DECISION | D5 — accepted-by-user]` The orchestrator proposes a tier as "Tier N — reply
  `go`, or name another tier"; it blocks and never proceeds on no reply.
- `[DECISION | D6 — accepted-by-user]` Add a new `/hotfix` slash command for Tier 0
  (mirrors `/quick-fix`), in addition to orchestrator auto-proposal.
- `[DECISION | D7 — accepted-by-user]` User under-tiering → the orchestrator warns but
  complies. Mid-flight discovery of ADR-worthy/architectural scope → the orchestrator pauses
  and re-proposes a higher tier to the user.
- `[DECISION | D8 — accepted-by-user]` Tier 1 == today's `/quick-fix` flow, the only change
  being the reviewer model downgraded to sonnet.
- `[DECISION | D9 — accepted-by-user]` Commit-message trailer `Change-Tier: trivial|small|full`
  (lowercase), added by the implementation-engineer on the final commit.
- `[RESEARCH | R1 — resolved]` The Agent/Task tool supports a per-invocation `model` override
  that takes precedence over an agent's frontmatter `model:`. A single `code-reviewer.md` can
  therefore be invoked with `model: sonnet` (Tier 1) or `model: opus` (Tier 2); the
  orchestrator supplies the model at spawn time.
- `[RESEARCH | R2 — resolved]` No auto-merge exists and none will be added. A CI workflow at
  `.github/workflows/ci.yml` already runs `just check` on every push and PR. Merging stays
  manual/user-confirmed. No branch protection.
- `[ASSUMPTION | accepted-by-user]` Scope is limited to the template's own config/docs files:
  `AGENTS.md`, `.claude/agents/*.md`, `.claude/commands/*.md`, a new `/hotfix` command, and
  the relevant READMEs. Other hooks are untouched; `just check` runs on every tier including
  Tier 0. No runtime/product code is in scope.
- `[ASSUMPTION | accepted-by-user]` Existing NFR mechanisms (secret-scan hook,
  prune-worktrees, SessionEnd cleanup) are untouched by this change.
