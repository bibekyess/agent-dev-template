---
title: Tiered execution pipeline with model-routing cost reductions — implementation plan
date: 2026-07-05
spec: specs/2026-07-05-tiered-pipeline.md
branch: feat/tiered-pipeline
---

# Plan — 2026-07-05 — Tiered execution pipeline with model-routing cost reductions

## Overview

Implement the ratified spec `specs/2026-07-05-tiered-pipeline.md` on branch
`feat/tiered-pipeline`. This is a **documentation/configuration-only** change to the
template's own agent definitions, slash commands, `AGENTS.md`, one new ADR, and a few
READMEs. No runtime/product code and **no shell hooks** are touched.

What is built:
- An explicit **three-tier execution model** (Tier 0 trivial/hotfix, Tier 1 small,
  Tier 2 full) documented as the single source of truth in `AGENTS.md` and referenced
  (not re-defined divergently) by the agent/command files.
- **Propose-and-confirm** tier selection in the orchestrator ("Tier N — reply `go`, or
  name another tier"), with block-on-no-reply and the two escalation rules
  (warn-and-comply for under-tiering; pause-and-re-propose for mid-flight architectural
  discovery).
- **Per-invocation reviewer model routing**: one `code-reviewer.md`, invoked with
  `model: sonnet` (Tier 1) or `model: opus` (Tier 2); orchestrator stays `opus`.
- A new **`/hotfix`** Tier 0 command mirroring `/quick-fix`.
- The **`Change-Tier: trivial|small|full`** commit trailer, added by the
  implementation-engineer on its final commit.
- One ADR recording the cross-cutting decision.

In scope: `AGENTS.md`, `.claude/agents/orchestrator.md`,
`.claude/agents/code-reviewer.md`, `.claude/agents/implementation-engineer.md`,
`.claude/commands/quick-fix.md`, `.claude/commands/review.md`,
`.claude/commands/hotfix.md` (new), `adr/2026-07-05-tiered-execution-model.md` (new),
`adr/README.md` (index entry), `plans/README.md`/`specs/README.md` (only if a tier
mention is warranted — see Step 11), `.github/PULL_REQUEST_TEMPLATE.md` (tier line).

Out of scope: any shell hook (`.claude/hooks/*`), `justfile`, `.github/workflows/ci.yml`
behavior, `requirements-engineer.md`, `researcher.md`, `technical-planner.md`
frontmatter/routing (their models are unchanged), and any auto-merge mechanism (none
exists; none is added — R2).

## Architecture & Design

**Single source of truth for the boundary (FR2, NFR-Consistency).** The tier boundary
("no new dependency/interface/schema change, nothing ADR-worthy") is defined **once** in
`AGENTS.md` → "Scale to the task". Every other file (orchestrator, `/hotfix`,
`/quick-fix`) **references** that section rather than restating the boundary in different
words. Where a file must name the flow, it names the tier and points to `AGENTS.md` for
the boundary definition.

**Model routing is data in two places that must agree (FR8, NFR-Consistency).**
- Agent frontmatter `model:` — the *default* model for that agent.
- `AGENTS.md` "Model routing" — the *documentation* of routing, which must match the
  frontmatter exactly, plus the note that the reviewer's *effective* model is set
  per-invocation by the orchestrator.
The only agent whose effective model differs from its frontmatter is the code-reviewer;
`AGENTS.md` must call that out explicitly so there is no drift.

**Per-invocation override (R1, FR7).** The orchestrator supplies the reviewer's model at
spawn time via the Agent/Task tool's per-invocation `model` field, which takes precedence
over `code-reviewer.md`'s frontmatter `model:`. There remains exactly **one** reviewer
file. See "Interface & Compatibility" for the syntax caveat the implementer must verify.

**No new dependencies.** Markdown/config only.

### Files and the specific edits each receives

| File | Change | Spec refs |
|---|---|---|
| `AGENTS.md` | Replace "Scale to the task" (2-tier) with 3-tier model; update "Model routing"; add `Change-Tier` trailer to "Git & commits" | FR1, FR2, FR8, FR9, FR10 |
| `.claude/agents/orchestrator.md` | Propose-and-confirm; block on no reply; escalation rules; per-invocation reviewer model override; note no 1M-context need; keep `model: opus` | FR3, FR4, FR5, FR6, FR7, edge cases |
| `.claude/agents/code-reviewer.md` | State effective model is set per-invocation by tier; keep single file (frontmatter `model:` stays `opus` as the default) | FR7, FR8 |
| `.claude/agents/implementation-engineer.md` | Add `Change-Tier` trailer on final commit; add Tier 0 (no reviewer, PR, one-word merge confirm) behavior | FR10, FR14, D9 |
| `.claude/commands/quick-fix.md` | Tier 1: reviewer invoked at `model: sonnet` | FR7, FR16, D8 |
| `.claude/commands/review.md` | Reviewer model supplied per-invocation by tier | FR7, FR8 |
| `.claude/commands/hotfix.md` (new) | Tier 0 command mirroring `/quick-fix` | FR17, D6, FR14 |
| `adr/2026-07-05-tiered-execution-model.md` (new) | Records the decision | FR-traceability |
| `adr/README.md` | Add index entry for the new ADR | adr/README convention |
| `.github/PULL_REQUEST_TEMPLATE.md` | Add a one-line tier note (minimal) | FR10 auditability |

## Architecture Decisions (ADRs)

### ADR to add — `adr/2026-07-05-tiered-execution-model.md`

This is a genuine cross-cutting decision (execution-model change + per-invocation model
routing with real alternatives). The implementer creates the file on the branch from the
content below, following `adr/TEMPLATE.md`. Set `status: Proposed` in the plan; the
implementation-engineer flips it to `Accepted` on the branch per `adr/README.md`
("merging a branch is the act of acceptance"). This ADR supersedes nothing.

```markdown
---
title: Tiered execution model with per-invocation reviewer model routing
date: 2026-07-05
status: Proposed        # Proposed | Accepted | Superseded
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
```

### Relevant existing ADRs for the implementer

None. `adr/README.md`'s index is currently empty ("none yet"), so there is no existing
`Accepted` ADR to respect or supersede. The implementer must still read `adr/TEMPLATE.md`
and `adr/README.md` (status lifecycle) before creating the new ADR.

## Implementation Steps (ordered)

Order rationale: define the single source of truth (`AGENTS.md`) first, then make the
agent/command files reference it, then the ADR + docs, then the gate. Each step is
traceable to the spec requirement(s) in brackets.

### Step 1 — `AGENTS.md`: replace "Scale to the task" with the three-tier model [FR1, FR2, FR11]

Replace the current "Scale to the task" section (lines 21–26) with a three-tier
description that is the **single source of truth** for the boundary. It must state:
- The boundary sentence (verbatim intent): a **trivial change** = a localized fix with no
  new dependency, interface, or schema change, and nothing ADR-worthy.
- **Tier 0 — Trivial / hotfix**: flow `branch → implement → just check → open PR →
  one-word user confirmation → merge`; **no code-reviewer stage**; all agent work on
  `sonnet`; reachable via `/hotfix` or orchestrator proposal.
- **Tier 1 — Small change** (`/quick-fix`): flow `implement → review` (spec + plan
  skipped); code-reviewer runs on `sonnet`; otherwise identical to today's `/quick-fix`.
- **Tier 2 — Full pipeline**: flow `spec → (research) → plan → implement → review`; `opus`
  on requirements, planning, and review; spec→plan→ADR chain intact.
- That spec/plan apply only to Tier 2; Tier 0 additionally produces no ADR [FR11].
- The guardrails that hold on **every** tier: `just check` before merge (incl. Tier 0),
  branch/worktree work only, `protect-primary-checkout.sh` unweakened, one-word user
  confirmation before merging the default branch [FR12–FR15].

Keep the file under ~150 lines (footer rule); write tersely in the file's existing bullet
style. Other files must **reference** this section for the boundary rather than restate it.

### Step 2 — `AGENTS.md`: update "Model routing" [FR8, FR9]

Replace the "Model routing" section (lines 28–32) so it documents, per agent, the model
that matches each agent file's `model:` frontmatter after this change, with **no
discrepancy**:
- researcher — `haiku`; implementation-engineer — `sonnet`; requirements-engineer and
  technical-planner — `opus` (Tier 2 only); orchestrator — `opus`.
- code-reviewer — state its frontmatter default is `opus`, but its **effective** model is
  set **per-invocation by the orchestrator**: `sonnet` for Tier 1 reviews, `opus` for
  Tier 2 reviews (Tier 0 has no reviewer). This is the one intentional
  frontmatter-vs-effective difference; naming it here prevents drift.

### Step 3 — `AGENTS.md`: add the `Change-Tier` trailer convention to "Git & commits" [FR10, D9]

In the "Git & commits" section (lines 70–74), add a bullet: every implementation commit
carries a `Change-Tier: trivial | small | full` git trailer (lowercase; one value,
matching the tier that ran); the implementation-engineer adds it on the final commit.

### Step 4 — `.claude/agents/orchestrator.md`: propose-and-confirm + escalation + reviewer override [FR3, FR4, FR5, FR6, FR7, edge cases]

Edit the orchestrator definition:
1. **Replace the "Scale to the task" paragraph** (line 33) with a **Tier selection**
   subsection instructing the orchestrator to, on every change request:
   - Classify the change against the AGENTS.md boundary and **propose exactly one tier**
     (0, 1, or 2) with a one-line rationale tied to that boundary.
   - Present it in the form **"Tier N — reply `go`, or name another tier"** [FR4, D5].
   - **Block and not proceed on no reply**; never silently default to the full pipeline
     [FR3, FR4, D5, edge case "no tier reply"].
   - If it cannot classify the change, **ask rather than guess** [edge case "ambiguous"].
   - Record the selected tier and pass it to every downstream stage (for the commit
     trailer and model routing) [FR5].
2. **Add escalation rules** [FR-edge, D7]:
   - **Under-tier override** (user picks a tier below the boundary, e.g. Tier 0 for a
     schema change): **warn but comply** — state the mismatch, then run the user-selected
     tier.
   - **Mid-flight discovery** (a Tier 0/1 change is found to need a new
     dependency/interface/schema change or an ADR): **pause and re-propose a higher tier**
     to the user; do not smuggle an architectural change through a lower tier.
3. **Per-tier flows** — align the existing "The pipeline" section so each tier's flow
   matches AGENTS.md:
   - Tier 2 = the full pipeline already described (unchanged behavior) [FR16, acceptance].
   - Tier 1 = implement → review (spec/plan skipped), reviewer invoked with **per-invocation
     `model: sonnet`** [FR7].
   - Tier 0 = branch → implement → `just check` → open PR → one-word user confirmation →
     merge, **no code-reviewer stage**, all agent work on `sonnet` [FR1, FR14].
4. **Reviewer model override** — in the review step(s), instruct that `code-reviewer.md` is
   spawned with a per-invocation `model` override: `model: sonnet` for Tier 1,
   `model: opus` for Tier 2 [FR7]. (See Interface & Compatibility for the exact-syntax
   caveat to verify.)
5. **Merge gate per tier** — update the "Gate the merge" rule: Tier 2 and Tier 1 merge on
   `reviewer-clean AND local-gate-clean`; Tier 0 merges on `local-gate-clean AND
   one-word-user-confirmation` (no reviewer) [FR15]. Keep "confirm with the user before
   merging the default branch".
6. **Keep `model: opus`** in frontmatter (unchanged) and add a **non-behavioral note**
   that the orchestrator does not require a 1M-context window (it passes only paths and
   short status, never artifact content) [FR6, acceptance].

### Step 5 — `.claude/commands/hotfix.md` (new): Tier 0 command [FR17, D6, FR14, FR12, FR13]

Create the file mirroring `quick-fix.md`'s shape and frontmatter style:
- Frontmatter `description:` = Tier 0 fast-path for trivial/hotfix changes — skips spec,
  plan, **and the code-reviewer stage**.
- Opening note: use only when the fix is trivial per the AGENTS.md boundary (reference it,
  don't restate divergently); anything outside that boundary must use `/quick-fix`
  (Tier 1) or the full pipeline (Tier 2).
- Steps: (1) confirm trivial per boundary; (2) create feature worktree
  `git worktree add .claude/worktrees/<slug> -b feat/<slug>` — work on a branch by
  construction, never the primary checkout on the default branch [FR13, edge case
  "protect-primary-checkout detection fails"]; (3) spawn `implementation-engineer` (on
  `sonnet`) with the worktree path + change description + a note this is Tier 0 (no
  spec/plan, add `Change-Tier: trivial` trailer); (4) engineer implements, runs
  `just check`, commits, pushes, opens a PR — **no reviewer** [FR14]; if `just check`
  fails the change does not merge and work continues on the branch [edge case "Tier 0 gate
  failure"]; (5) after `just check`/CI is green, require a **one-word user confirmation**
  before merging the default branch, then merge [FR14, D3].
- Return the PR URL once open.

### Step 6 — `.claude/commands/quick-fix.md`: Tier 1 reviewer on sonnet [FR7, FR16, D8]

Edit step 5 ("Spawn `code-reviewer` on the PR…") so it spawns `code-reviewer` with a
per-invocation **`model: sonnet`** override. Optionally add a one-line note that this is
the **Tier 1** flow, identical to before except the reviewer model. Do not otherwise
change the flow (D8).

### Step 7 — `.claude/commands/review.md`: reviewer model supplied per-invocation [FR7, FR8]

Edit so the review command states that the reviewer's **model is supplied per-invocation
by tier** — `model: sonnet` for Tier 1, `model: opus` for Tier 2 — by the orchestrator at
spawn time, rather than fixed by the agent's frontmatter. Keep the existing pass list and
loop-until-APPROVED behavior.

### Step 8 — `.claude/agents/code-reviewer.md`: note per-invocation model [FR7, FR8]

Keep exactly **one** reviewer file and keep its frontmatter `model: opus` as the default.
Add a short note (near the top or in "Rules") that the **effective** model is set
per-invocation by the orchestrator — `sonnet` for Tier 1, `opus` for Tier 2 — so the
frontmatter is only the default. Do **not** create any `code-reviewer-lite.md` or second
variant [acceptance: exactly one reviewer].

### Step 9 — `.claude/agents/implementation-engineer.md`: trailer + Tier 0 behavior [FR10, D9, FR14]

1. **Trailer** — in "Phase 5 — Commit, push, PR" (step 8) and/or "Rules", require the
   final commit to carry a lowercase `Change-Tier: trivial|small|full` trailer matching
   the tier the orchestrator passed. State the orchestrator supplies the tier.
2. **Tier 0 behavior** — add that when invoked for Tier 0 (no spec/plan): implement the
   described change, run `just check`, commit with `Change-Tier: trivial`, push, open a PR;
   there is **no reviewer stage**, and the orchestrator awaits a one-word user merge
   confirmation before the default branch is merged [FR14]. Do not weaken the existing
   "work only in the worktree / never the default branch" rule.

### Step 10 — `adr/2026-07-05-tiered-execution-model.md` (new) + `adr/README.md` index

Create the ADR from the content in "Architecture Decisions" above (`status: Proposed` in
the plan; the implementer flips to `Accepted` on the branch per `adr/README.md`). Add an
index entry under `adr/README.md` "## Index" replacing "(none yet — add your first ADR
here)" with a link/line for `2026-07-05-tiered-execution-model.md`.

### Step 11 — Minimal docs: PR template + READMEs [FR10 auditability, FR17]

- `.github/PULL_REQUEST_TEMPLATE.md`: add a single line near "What & why" noting the PR's
  tier / that the commit carries a `Change-Tier` trailer. Keep it one line — do not
  restructure the template.
- `specs/README.md` / `plans/README.md`: only add a mention if needed to state that
  specs/plans are produced for **Tier 2** (and Tier 1/0 skip them). Keep to at most one
  sentence each; skip entirely if the existing text already implies it and a change would
  be redundant (prefer the single source of truth in `AGENTS.md`). State the decision made
  in the PR description either way.

### Step 12 — Run the quality gate [FR12]

Run `just check` in the worktree and confirm every recipe passes. This is the final step
before commit for this change itself.

## Interface & Compatibility

Observable "contracts" here are the slash-command surface and agent-invocation behavior:
- `/spec`, `/plan`, `/implement`, `/review` MUST continue to work and map to the Tier 2
  flow — **do not change their behavior** beyond `review.md`'s per-invocation model note
  [FR16].
- `/quick-fix` MUST continue to work and map to Tier 1; the **only** behavioral change is
  the reviewer model (`sonnet`) [FR16, D8].
- `/hotfix` is **new** and additive — no existing command changes to accommodate it
  [FR17].
- The orchestrator's `tools:` frontmatter already lists `Agent(code-reviewer)` etc.; no
  tool grant changes are required.

**Per-invocation `model` override — verify the exact syntax (flagged).** Spec R1 asserts
the Agent/Task tool supports a per-invocation `model` that overrides the agent's
frontmatter `model:`. The implementer MUST confirm the precise field name/spelling the
host uses when the orchestrator spawns a subagent (e.g. how `model: sonnet` /
`model: opus` is passed alongside the subagent type) and write the orchestrator's
instructions using that exact form. If the host does not accept a per-invocation model
override, this is a **NEEDS RESEARCH / stop-and-escalate** condition — do **not** fall back
to creating a second reviewer file (D2 rejects that); pause and re-confirm with the
orchestrator/user, because the whole FR7 routing depends on R1 holding.

## Data / Migration Notes

None. No storage, schema, migration, or data is touched. This is docs/config only.

## Test Strategy

This repo is documentation/config; verification is (a) the quality gate and (b)
file-inspection against the spec's acceptance criteria. There is no unit-test surface to
add. Verify each of the following (map to the spec's Acceptance Criteria and the FRs):

**(a) Quality gate**
- Run `just check` in the worktree — all recipes (`format`, `lint`, `typecheck`, `test`)
  pass (currently no-ops, must stay green) [FR12].

**(b) File-inspection checks** (use `grep`/`Read`; run from the worktree root):
1. **Three tiers named in AGENTS.md** — `grep -n "Tier 0" AGENTS.md`, `"Tier 1"`,
   `"Tier 2"` all present in "Scale to the task"; boundary sentence present once [FR1, FR2].
2. **AGENTS.md routing matches frontmatter** — the "Model routing" section names
   researcher=haiku, implementation-engineer=sonnet, requirements-engineer=opus,
   technical-planner=opus, orchestrator=opus, and states code-reviewer effective model is
   per-invocation (sonnet T1 / opus T2). Cross-check each agent file's `model:` frontmatter
   with `grep -n "^model:" .claude/agents/*.md` — no discrepancy [FR8, FR9].
3. **Exactly one reviewer file** — `ls .claude/agents/ | grep -i review` returns exactly
   `code-reviewer.md`; no `*-lite*` / second variant [FR7, acceptance].
4. **Orchestrator propose-and-confirm** — `grep` for the phrase form "reply `go`" / "name
   another tier", for a block-on-no-reply statement, and that the silent full-pipeline
   default is gone [FR3, FR4, D5]. Confirm `model: opus` still in frontmatter and the
   "no 1M-context" note is present [FR6].
5. **Orchestrator reviewer override** — Tier 1 review step names `model: sonnet`, Tier 2
   names `model: opus` [FR7].
6. **Escalation rules present** — `grep` orchestrator (and/or AGENTS.md) for "warn"/"comply"
   (under-tier) and "pause"/"re-propose" (mid-flight) [D7, edge cases].
7. **Tier 0 flow (orchestrator + `/hotfix`)** — contains `branch → implement → just check →
   open PR → one-word user confirmation → merge`, **no reviewer stage**, all agent work on
   `sonnet`; requires `just check` before merge and branch/worktree work [FR1, FR14, FR13].
8. **Tier 1 flow (`/quick-fix`)** — implement → review with reviewer at `model: sonnet`;
   otherwise unchanged [FR7, D8].
9. **Trailer** — implementation-engineer requires lowercase
   `Change-Tier: trivial|small|full` on the final commit; AGENTS.md "Git & commits"
   documents it; PR template references it [FR10, D9].
10. **Hooks unchanged** — `git -C <worktree> diff main -- .claude/hooks/` is **empty**
    (no hook disabled, weakened, or modified) [FR13, acceptance].
11. **Tier 2 intact** — spec→(research)→plan→implement→review with opus on requirements,
    planning, review; spec→plan→ADR chain intact; `/spec` `/plan` `/implement` `/review`
    still map to Tier 2 [FR16, acceptance].
12. **New `/hotfix` exists and maps to Tier 0** — file present, mirrors `/quick-fix` shape
    [FR17].
13. **ADR present & well-formed** — `adr/2026-07-05-tiered-execution-model.md` follows
    `adr/TEMPLATE.md` (Context/Decision/Alternatives/Consequences), lists the three
    rejected alternatives (no-PR hotfix, two reviewer files, downgrading orchestrator),
    and is indexed in `adr/README.md`. Status `Accepted` on the branch before merge.

Scenario coverage mapping (spec edge cases → where verified): no-reply-block (check 4),
under-tier warn-comply and mid-flight re-propose (check 6), Tier 0 gate failure
(documented in `/hotfix` step, check 7), primary-checkout protection not sole guard
(check 7 branch-by-construction + check 10 hook unchanged).

## Risk & Sequencing

**Sequencing.** Step 1 (AGENTS.md single source of truth) precedes Steps 4–8 because those
files reference it. Step 10 (ADR) can be authored any time but its `Accepted` flip and the
final commit trailer come at commit time. Step 12 (`just check`) is last.

**Risks & mitigations.**
- *Per-invocation `model` override syntax may differ from R1's assumption.* Mitigation:
  the implementer verifies the exact host syntax before writing the orchestrator's review
  steps; on failure, stop-and-escalate (do NOT add a second reviewer file). See Interface
  & Compatibility. (Primary risk.)
- *Drift between AGENTS.md routing and agent frontmatter.* Mitigation: check 2 cross-checks
  every `model:` line against the doc; the code-reviewer difference is explicitly documented.
- *Accidentally restating the boundary divergently across files.* Mitigation: reference the
  AGENTS.md "Scale to the task" section from other files instead of re-defining (FR2,
  NFR-Consistency).
- *Touching a hook.* Mitigation: hooks are out of scope; check 10 asserts an empty diff
  under `.claude/hooks/`.
- *AGENTS.md exceeding ~150 lines.* Mitigation: keep the tier text terse; prune redundant
  wording rather than pad.
