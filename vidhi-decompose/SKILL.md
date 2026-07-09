---
name: vidhi-decompose
description: Break a plan, spec, or PRD into independently-grabbable issues on the project issue tracker using tracer-bullet vertical slices. Use when user wants to convert a plan into issues, create implementation tickets, or break down work into issues.
---

# Decompose

Break a plan into independently-grabbable issues using vertical slices (tracer bullets).

The issue tracker and triage label vocabulary should have been provided to you — run `/vidhi-init` if not.

## Process

### 1. Gather context

Work from whatever is already in the conversation context. If the user passes an issue reference (issue number, URL, or path) as an argument, fetch it from the issue tracker and read its full body and comments.

### 2. Explore the codebase (optional)

If you have not already explored the codebase, do so to understand the current state of the code. Issue titles and descriptions should use the project's domain glossary vocabulary, and respect ADRs in the area you're touching.

### 3. Draft vertical slices

Break the plan into **tracer bullet** issues. Each issue is a thin vertical slice that cuts through ALL integration layers end-to-end, NOT a horizontal slice of one layer.

Slices may be 'HITL' or 'AFK'. HITL slices require human interaction, such as an architectural decision or a design review. AFK slices can be implemented and merged without human interaction. Prefer AFK over HITL where possible.

<vertical-slice-rules>
- Each slice delivers a narrow but COMPLETE path through every layer (schema, API, UI, tests)
- A completed slice is demoable or verifiable on its own
- Each slice is sized to fit in a single fresh context window
- Prefer many thin slices over few thick ones
</vertical-slice-rules>

**Wide refactors are the exception to vertical slicing.** A **wide refactor** is one mechanical change — rename a column, retype a shared symbol — whose **blast radius** fans across the whole codebase, so a single edit breaks thousands of call sites at once and no vertical slice can land green. Don't force it into a tracer bullet; sequence it as **expand–contract**. First expand: add the new form beside the old so nothing breaks. Then migrate the call sites over in batches sized by blast radius (per package, per directory), each batch its own task blocked by the expand, keeping CI green batch to batch because the old form still exists. Finally contract: delete the old form once no caller remains, in a task blocked by every migrate batch. When even the batches can't stay green alone, keep the sequence but let them share an integration branch that all block a final integrate-and-verify task — green is promised only there. (Use `sutra_impact`/`sutra_refs` to size the blast radius when sketching the batches.)

### 4. Quiz the user

Present the proposed breakdown as a numbered list. For each slice, show:

- **Title**: short descriptive name
- **Type**: HITL / AFK
- **Needs plan**: yes / no — whether the task warrants vidhi-plan before implementation
- **Blocked by**: which other slices (if any) must complete first
- **User stories covered**: which user stories this addresses (if the source material has them)

**Needs plan** heuristic: tag `needs-plan` when the task touches multiple files, has integration points with other tasks, or requires non-obvious sequencing. Skip for single-file changes, config tweaks, or tasks where the acceptance criteria are the plan.

Ask the user:

- Does the granularity feel right? (too coarse / too fine)
- Are the dependency relationships correct?
- Should any slices be merged or split further?
- Are the correct slices marked as HITL and AFK?

Iterate until the user approves the breakdown.

### 5. Propose review checkpoints

Group the approved slices into **review checkpoints** — points where work is reviewed before continuing. Each checkpoint is a set of slices that, when complete, leave the system in a coherent state where a reviewer won't flag "missing" things that are simply upcoming work.

Good checkpoint boundaries:
- After foundational/mechanical changes (renames, migrations, config) that everything else builds on
- After a set of core features that are independently complete
- After integration/cap slices that tie subsystems together

Present the proposed checkpoints to the user and confirm. For each checkpoint, show which slices are included and why this is a stable review point.

**Sutra tending**: if the repo is sutra-governed (`.sutra/rules.toml` exists or vidhi-sutra-seed ran against this PRD), decide which checkpoints are also **vidhi-sutra-tend** moments, and say so in those review tasks' bodies. Heuristics:

- The **first checkpoint after foundational slices** is almost always one — module interiors exist for the first time (and can be constrained while still small), and FCA has data for the initial convention triage.
- A checkpoint where a **track's code has settled** (its slices complete, before the next track builds on them) is a natural tend moment for the structure that track grew.
- Cross-check `.sutra/rules.toml` for deferred `# TRIGGER:` blocks: every trigger naming a checkpoint or review event must map to an actual published task, or the deferred constraint is orphaned. Where a trigger names a task you're about to create, reference the real task id back into the rules file (or flag it for the seed/tend pass to update).

Not every checkpoint needs tending — a checkpoint reviewing pure behavior changes inside already-constrained structure doesn't. Name the tend moments deliberately rather than blanketing every review task.

When publishing (step 6), create a **review task** for each checkpoint:
- Type: HITL, status: `ready-for-human`
- Title: "5x-review-N: review {checkpoint description}"
- Depends on all slices in the checkpoint
- Tag with `review`
- Body describes what the reviewer should focus on at this stage

**Ordering**: publish review tasks interleaved with implementation slices so the dependency graph reads in execution order. Concretely: publish checkpoint 1's slices, then checkpoint 1's review task, then checkpoint 2's slices, then checkpoint 2's review task, etc. This way the issue list reads top-to-bottom as the work will actually proceed.

Slices in subsequent checkpoints depend on the preceding review task — the review is a quality gate that must pass before the next wave begins.

### 6. Publish the issues to the issue tracker

For each approved slice, publish a new issue to the issue tracker. Use the issue body template below. If the parent PRD belongs to a yojana arc, publish all slices and review tasks to the same arc with `arc_phase: implement`.

Set the status directly based on what the task needs:

| Task is | Status |
|---|---|
| AFK and unblocked | `ready-for-agent` |
| HITL (design decision, review, grilling) | `ready-for-human` |
| Blocked by another task | `ready-for-agent` or `ready-for-human` (the dependency graph handles blocking) |

Tag tasks with `needs-plan` when marked as such during the quiz step.

Publish issues in dependency order (blockers first) so you can reference real issue identifiers in the "Blocked by" field.

<issue-template>
## Parent

A reference to the parent issue on the issue tracker (if the source was an existing issue, otherwise omit this section).

## What to build

A concise description of this vertical slice. Describe the end-to-end behavior, not layer-by-layer implementation.

## Acceptance criteria

- [ ] Criterion 1
- [ ] Criterion 2
- [ ] Criterion 3

## Blocked by

- A reference to the blocking ticket (if any)

Or "None - can start immediately" if no blockers.

</issue-template>

Do NOT close or modify any parent issue.
