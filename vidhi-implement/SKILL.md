---
name: vidhi-implement
description: Execute a yojana task using the appropriate methodology. Takes a task ID, pulls context from yojana, selects the execution method (TDD for features, diagnose for bugs, architecture for refactoring), runs it, and reports results back.
---

# Implement a Yojana Task

## Entry

Requires a yojana task ID (e.g., `project/N`).

## Workflow

### 1. Pull context

```
yojana_context task="<task-id>" shape="working"
```

This gives you: acceptance criteria, decisions, implementation plan, conversation history, and neighbor context (dependencies, related tasks).

### 2. Check for implementation plan

If the context includes an `implementation_plan`, use it as your execution roadmap — it has the file map, step sequence, and code sketches from vidhi-plan. Follow the steps in order. Don't re-derive the approach.

If there's no implementation plan, proceed with method selection below. The task may not have gone through vidhi-plan (and that's fine for smaller tasks).

### 3. Select method

Based on the task's category and tags:

| Category/Tag | Method |
|---|---|
| `enhancement` | vidhi-tdd |
| `bug` | vidhi-diagnose |
| tag `refactor` or `architecture` | vidhi-architecture |
| unclear | Ask the user |

### 4. Execute

Run the selected methodology skill. The task's acceptance criteria serve as the behavior spec — you don't need to re-confirm "which behaviors to test?" if the criteria are well-written.

### 5. Report progress

During execution:
- Each meaningful milestone → `yojana_task action=comment id="<task-id>" text="<progress>" author="agent"`
- Mark acceptance criteria done as they're satisfied → `yojana_task action=update id="<task-id>" acceptance_criteria=[...]`
- Record design decisions discovered during implementation → update `decisions` field

### 6. Complete

When all acceptance criteria are green:
```
yojana_task action=update id="<task-id>" status="done"
```
this needs to be updated because there should now be a new field where the git commits
and other info can be put.

If the task can't be completed (blocker found, scope unclear):
```
yojana_task action=comment id="<task-id>" text="<what's blocking>" author="agent"
yojana_task action=update id="<task-id>" status="needs-info"
```
