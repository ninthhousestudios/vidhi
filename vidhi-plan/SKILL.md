---
name: vidhi-plan
description: Plan the implementation of a single yojana task. Explores the codebase, maps files, sequences steps with code sketches, and writes the plan to the task's implementation_plan field. Use after decompose, before implement.
---

# Plan a Yojana Task

## Entry

Requires a yojana task ID (e.g., `project/N`).

## Workflow

### 1. Pull context

```
yojana_context task="<task-id>" shape="working"
```

Read the acceptance criteria, decisions, description, and neighbor context. If acceptance criteria are missing or vague, stop and set the task to `needs-info` with a comment explaining what's unclear.

**Re-validate the description against current state.** Task descriptions are written at creation time and rot: an architecture decision made after the task was filed can invalidate its assumptions, and dependencies move under multi-day-old tasks (reflect 2026-07-02: ~10 swisseph-rs planning tasks each independently re-corrected the same stale stateful-caching assumption; vidhi/7 executed against a dependency that had moved the same day). Check the description's claims against the code as it is now; note corrections in the plan. If a landmark decision has invalidated many descriptions at once, propose a one-time backlog sweep instead of letting every future planning pass rediscover the same correction.

### 2. Explore the codebase in a precise, targeted way

Before planning anything, go find the code. Read the files that this task will touch. Understand the interfaces, the types, the patterns already in use.

What to look for:
- Files that implement the feature area (or adjacent features)
- Test files and testing patterns used in this project
- Types, interfaces, or schemas this task will interact with
- Configuration or wiring that will need to change

Don't skim — read enough to write code against these interfaces. A plan built on guesses about the codebase is worse than no plan.

### 3. Check neighbors

Review the 1-hop neighbors from the context bundle. Look for:
- **Upstream tasks** (dependencies): are they done? If not, what assumptions are you making about their output?
- **Sibling tasks** that touch the same files: will your plan conflict? Call out shared files explicitly.
- **Downstream tasks** that depend on this one: does your plan produce what they need?

If you find a conflict or dependency issue, comment on the task and flag it to the user before continuing.

### 4. Write the plan

**Types-first gate (Rust/Dart).** If the task creates a non-trivial new unit — a module or subsystem with a real data model — suggest `vidhi-types-first` in one line before writing the plan. Its approved type skeleton becomes this plan's code sketches: signatures land in the Steps section already reviewed, and implementation codes against a contract instead of a guess. Josh decides; don't start it unbidden.

Structure:

```markdown
## Approach

1-3 sentences: what's the strategy? Why this approach over alternatives?

## Files

- **Create**: `path/to/new-file.ts` — what it's responsible for
- **Modify**: `path/to/existing.ts` (lines ~50-80) — what changes and why
- **Test**: `path/to/test-file.test.ts` — what's being tested

## Steps

### Step 1: [description]

What to do, why, and a code sketch showing the shape of the change:

```lang
// sketch — types, function signatures, key logic
// not full implementation, but enough to code against
```

### Step 2: [description]
...

## Acceptance criteria coverage

- [x] "criteria text" → Step N
- [x] "criteria text" → Steps N, M
```

### Plan quality checks

Before writing it to yojana, verify:

- **Every acceptance criterion** maps to at least one step. If one doesn't, you missed something — add a step.
- **Every step** has a code sketch or concrete description. "Add error handling" is not a step. Show what error, what handling.
- **File paths are real.** Don't guess paths — you explored the codebase in step 2. Use the actual paths you found.
- **No placeholders.** No "TBD", "TODO", "implement appropriately", "similar to step N". If you can't be specific, you didn't explore enough — go back to step 2.
- **Steps are ordered by dependency**, not by file. If step 3 depends on step 1's output, that should be obvious from the sequence.

### 5. Save the plan

```
yojana_task action=update id="<task-id>" implementation_plan="<plan>"
```

### 6. User review

Tell the user the plan is written and ask them to review before implementation:

> "Plan written to `<task-id>`. [brief summary of approach and step count]. Want to review before we implement?"

Do not proceed to vidhi-implement until the user approves or says to go ahead.
