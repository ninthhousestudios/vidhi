---
name: vidhi-premortem
description: Stress-test a decomposition before implementation begins. Reviews the full set of slices for coverage gaps, integration risks, dependency problems, and timeline traps. Use after vidhi-decompose, before picking up tasks.
---

# Pre-Mortem

> Imagine we implemented this plan exactly as decomposed and it failed. Why did it fail?

Run this after decompose has published issues and before anyone starts implementing. The goal is to catch problems that no single task can see: gaps between slices, integration risks, wrong ordering, missing error handling.

## Entry

Accepts a project slug or parent task ID. Pulls all tasks from that scope via `yojana_query`.

If not provided, ask the user which project or parent task to review.

## Process

### 1. Load the decomposition

Query for the tasks:

```
yojana_query project="<slug>" status="needs-triage"
```

or if given a parent task, query the project and filter by tasks that refine it. Also pull the PRD/parent task body for reference.

For each task, read its title, description, acceptance criteria, slice type (HITL/AFK), and edges (dependencies). Build a mental map of the full decomposition.

### 2. Determine scope posture

Before reviewing, decide the posture. Three modes:

| Mode | When | Posture |
|------|------|---------|
| **Hold scope** | Bug fixes, refactors, most work | Make it bulletproof within accepted scope |
| **Reduce scope** | Decomposition has 8+ tasks, or tasks feel overbuilt | Strip to essentials. What's the minimum that ships value? |
| **Expand scope** | Greenfield, user said "go big" | What's missing that would make this great? |

Auto-detect if the user doesn't specify. Default to hold scope. Announce the posture before starting the review.

### 3. Coverage review

Check every requirement in the PRD against the decomposed tasks:

- Does every user story / requirement map to at least one task's acceptance criteria?
- Are there requirements that fell through the cracks during decomposition?
- Are there tasks that don't trace back to any requirement? (scope creep)

### 4. Integration seams

Look at the boundaries between tasks:

- When task A produces something that task B consumes, is the interface explicit in both tasks' acceptance criteria?
- Are there integration points that no task owns? ("Task 1 creates the API, task 3 calls the API, but nobody tests them together")
- If tasks touch the same files, are the boundaries clear enough to avoid conflicts?

### 5. Dependency walk

Review the dependency graph:

- Are there hidden dependencies? (Task B can't actually start until task A is done, but no edge exists)
- Are there unnecessary dependencies? (Tasks marked as blocked that could actually run in parallel)
- Is the critical path obvious? Which task, if delayed, delays everything?

### 6. Temporal walk

Walk through the implementation in dependency order. For each phase, ask:

| Phase | Questions |
|-------|-----------|
| **First task** | What blocks the first meaningful change? Are prerequisites actually met? |
| **Middle tasks** | Which integration points are untested until late? What fails silently when components connect? |
| **Final tasks** | What "should be quick" but historically isn't? What's left untested until everything is assembled? |

Look for late integration risks — things that won't surface until most of the work is done and are expensive to fix.

### 7. Error paths

For tasks that involve external calls (APIs, databases, file I/O, LLM calls):

- Is error handling covered in the acceptance criteria, or is it assumed?
- What happens when the external call fails? Is that behavior specified anywhere?
- If multiple tasks depend on the same external service, is the failure mode consistent across them?

Skip this section if no tasks involve external calls.

### 8. Test coverage

For each task:

- Do code-change tasks have testing expectations in their acceptance criteria?
- Are integration-level tests planned, or does every task only test in isolation?
- Is there a task (or criteria within a task) for end-to-end verification after all slices land?

### 9. Verdict

| Verdict | Meaning | Action |
|---------|---------|--------|
| **Ready** | No significant issues found | Proceed to implementation |
| **Concerns** | Issues found but addressable | Present concerns, fix before implementing |
| **Not ready** | Fundamental problems with the decomposition | Needs redesign — re-run vidhi-decompose after fixing |

### 10. Report

Present findings to the user, organized by severity:

- **Gaps**: requirements with no covering task
- **Integration risks**: seams between tasks that nobody owns
- **Dependency issues**: missing or incorrect edges
- **Timeline traps**: things that will bite late
- **Missing error handling**: unspecified failure modes

For each finding, recommend a fix: add a task, add acceptance criteria to an existing task, add a dependency edge, or split/merge tasks.

If the verdict is "Ready" or "Concerns" (after fixes), comment on the parent/PRD task that the pre-mortem passed:

```
yojana_task action=comment id="<parent-id>" text="Pre-mortem: <verdict>. <1-2 sentence summary>" author="agent"
```

Do not proceed to vidhi-plan or vidhi-implement until the user acknowledges the findings.
