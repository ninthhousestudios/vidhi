# vidhi

Engineering methodology skills for Claude Code agents. Forked from [mattpocock/engineering](https://github.com/mattpocock/claude-code-skills) and adapted for the [manas](https://github.com/josharp/manas) ecosystem.

## Skills

### Grounding (phase 1)

| Skill | Purpose |
|---|---|
| `vidhi-init` | Bootstrap project context docs (domain, issue-tracker, triage-labels) |
| `vidhi-domain` | Establish/refine CONTEXT.md and ADRs — domain vocabulary |
| `vidhi-survey` | Big-picture assessment of current state |

### Planning (phase 2)

| Skill | Purpose |
|---|---|
| `vidhi-prd` | Feature idea → PRD (user stories, modules, test strategy) |
| `vidhi-decompose` | PRD → vertical slices, published to yojana |
| `vidhi-triage` | Route each task: agent/human/needs-info/wontfix |

### Execution (phase 3)

| Skill | Purpose |
|---|---|
| `vidhi-implement` | Yojana-aware dispatcher — pulls task, selects method, reports back |
| `vidhi-tdd` | Red-green-refactor loop (standalone or called by vidhi-implement) |
| `vidhi-diagnose` | Bug investigation — hypothesize, isolate, confirm |
| `vidhi-architecture` | Refactoring and structural improvement |

### Verification (phase 4)

| Skill | Purpose |
|---|---|
| `vidhi-release-review` | Multi-pass review at release or checkpoint moments — pack + parallel architecture/review subagents + synthesis |

## Usage with yojana

vidhi provides the *methodology* (opinions about how to plan and execute). [yojana](../yojana/) provides the *grammar* (task schema, state machine, dependencies, context shapes).

### Planning a new feature

```
1. vidhi-init          → bootstrap context docs in your project
2. vidhi-domain        → ground domain vocabulary (CONTEXT.md)
3. vidhi-prd           → write the PRD from your feature idea
4. vidhi-decompose     → break PRD into yojana tasks with dependencies
5. vidhi-triage        → label each task for routing
```

### Executing tasks

```
yojana_ready project="my-project"    → find next unblocked task
vidhi-implement project/N            → pull context, execute with TDD/diagnose/architecture, report back
```

Or standalone (no yojana):
```
vidhi-tdd                            → ad-hoc TDD on whatever you're building
vidhi-diagnose                       → investigate a bug without a tracked task
```

## Installation

Add each skill directory to your Claude Code slash commands, or reference them via the skill index in `~/.claude/skill-index.md`.

## License

MIT — see [LICENSE](LICENSE). Originally by Matt Pocock; adapted by Josh Harper.
