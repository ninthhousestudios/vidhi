# vidhi

Engineering methodology skills for Claude Code agents. Forked from [mattpocock/engineering](https://github.com/mattpocock/claude-code-skills) and adapted for the [manas](https://github.com/ninthhousestudios/manas) ecosystem.

## Skills

### Grounding

| Skill | Purpose |
|---|---|
| `vidhi-init` | Bootstrap project context docs (domain, issue-tracker, triage-labels) |
| `vidhi-domain` | Establish/refine CONTEXT.md and ADRs — domain vocabulary |
| `vidhi-survey` | Big-picture assessment of current state |

### Design

| Skill | Purpose |
|---|---|
| `vidhi-brainstorm` | Collaborative design exploration — structured dialogue to turn ideas into designs |
| `vidhi-prd` | Synthesize conversation context into a PRD (user stories, modules, test strategy) |

### Planning

| Skill | Purpose |
|---|---|
| `vidhi-decompose` | PRD → vertical slices, published to yojana with status and planning tags |
| `vidhi-premortem` | Stress-test a decomposition before implementation — coverage, integration, timeline risks |
| `vidhi-plan` | Per-task implementation planning — file map, step sequence, code sketches → yojana `implementation_plan` |

### Execution

| Skill | Purpose |
|---|---|
| `vidhi-implement` | Yojana-aware dispatcher — pulls task, follows plan if present, selects method, reports back |
| `vidhi-tdd` | Red-green-refactor loop (standalone or called by vidhi-implement) |
| `vidhi-diagnose` | Bug investigation — hypothesize, isolate, confirm |
| `vidhi-deepen` | Refactoring and structural improvement |

### Verification

| Skill | Purpose |
|---|---|
| `vidhi-review` | Per-task code review against acceptance criteria — inline, codex, or brief mode |
| `vidhi-release-review` | Multi-pass project review at release or checkpoint — pack + parallel subagents + synthesis |

## Usage

vidhi provides the *methodology* (opinions about how to plan and execute). [yojana](https://github.com/ninthhousestudios/yojana/) provides the *grammar* (task schema, state machine, dependencies, context shapes). [sutra](https://github.com/ninthhousestudios/sutra/) provides *code intelligence* (impact analysis, hotspots, dead code).

### Full pipeline: idea → shipped feature

```
vidhi-brainstorm      → explore the design space, one question at a time
  ↓
vidhi-domain          → sharpen vocabulary, update CONTEXT.md (optional — skip if domain is clear)
  ↓
vidhi-prd             → capture the design as a PRD
  ↓
vidhi-decompose       → break PRD into yojana tasks with dependencies and planning tags
  ↓
vidhi-premortem       → stress-test the decomposition for gaps and risks
  ↓
[per wave of unblocked tasks]
  vidhi-plan          → plan tasks tagged needs-plan (skip for simple tasks)
  vidhi-implement     → execute with TDD/diagnose/architecture, report back
  vidhi-review        → review completed tasks
```

Not every project needs every step. Small features can skip brainstorm and go straight to PRD. Bug fixes can skip everything and go straight to `vidhi-diagnose`. Use judgment.

### Planning in waves

Don't plan all tasks upfront. Plan and implement in waves following the dependency graph:

1. After decompose + premortem, plan the first batch of unblocked `needs-plan` tasks
2. Implement them (simple tasks without `needs-plan` go straight to implement)
3. Plan the next batch — now informed by what you learned implementing the first wave
4. Repeat until done

Plans also serve as session bookmarks. If a task spans sessions, the plan tracks which steps are complete so the next session picks up where you left off.

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

### Reviewing code

vidhi has two review skills for different scopes.

#### Per-task review (`vidhi-review`)

Review code changes for one or more yojana tasks. Checks the diff against the task's acceptance criteria, project invariants, and blast radius from sutra.

```
vidhi-review chitta/32                           → inline review (claude, default)
vidhi-review chitta/32 --reviewer codex          → dispatch to codex
vidhi-review chitta/32 --reviewer brief          → produce review brief for any external model
vidhi-review chitta/32 chitta/33                 → review multiple related tasks together
```

**Reviewers:**

| Reviewer | What happens |
|---|---|
| `claude` (default) | Reviews inline in the current session. Produces findings in conversation, updates yojana. |
| `codex` | Calls codex companion script directly with task context as focus text. Records codex output on the yojana task. |
| `brief` | Writes a self-contained directory to `/tmp/review-brief-<project>-<date>/` with instructions, task context, diff, impact analysis, and project context. Feed it to any model. |

**Diff resolution** (automatic, in priority order):

1. `git:commit` context refs on the task(s) — uses recorded SHAs
2. User-specified range — `vidhi-review chitta/32 --range main..feature`
3. Asks if neither is available

**Outputs:**

- Findings in YAML schema (same as vidhi-release-review — interchangeable)
- Verdict: `approve` / `approve with notes` / `request changes`
- Yojana comment on the task with verdict and summary
- Follow-up tasks filed for new work surfaced during review (with user confirmation)

#### Release review (`vidhi-release-review`)

Full project audit at release or checkpoint moments. Builds a review pack with code intelligence, runs two parallel subagent passes (architecture + code review), and synthesizes findings into an action plan.

```
vidhi-release-review                             → review current project at HEAD
vidhi-release-review --scope wm-1..wm-12         → review a specific commit range
```

**Process:**

1. Verify project state (clean working tree, confirm scope with user)
2. Build review pack at `/tmp/review-pack-<project>-<date>/`
   - `00-meta.md` — scope, versions, review type
   - `10-context/` — README, CLAUDE.md, CONTEXT.md, ADRs, manifests
   - `20-code-intel/SUMMARY.md` — curated hotspots, file health, filtered dead code (from sutra)
   - `30-verification/SUMMARY.md` — build, test, lint, format results
3. Curate code-intel SUMMARY (don't dump raw sutra JSON — reviewer attention is the constraint)
4. Run verification (language-appropriate build/test/lint/fmt)
5. Launch two parallel subagents:
   - **Architecture pass** — vidhi-deepen skill, produces deepening candidates
   - **Code review pass** — reviewer-prompt.md, produces findings with YAML schema
6. Synthesize both reviews into a single action plan with fix-order waves
7. Triage findings to yojana (with user confirmation)

**Outputs** (three files in `docs/reviews/`):

- `<date>-<scope>-architecture.md` — architecture pass
- `<date>-<scope>-review.md` — code review pass
- `<date>-<scope>.md` — synthesis (read this first)

**Checkpoint mode:** for mid-upgrade reviews, substitute the verdict with `continue as planned` / `continue with adjustments` / `pause and address issues`.

**Multi-model support:** the review pack and reviewer prompt are model-agnostic. Hand the pack directory + `reviewer-prompt.md` to codex, opencode, gemini, or any model. The synthesis step reconciles N reviews instead of 2.

### Choosing between review skills

| Situation | Skill |
|---|---|
| Just finished a task, want to check correctness | `vidhi-review` |
| Reviewing 2-3 related tasks together | `vidhi-review` (multi-task) |
| Want a second opinion from codex on a task | `vidhi-review --reviewer codex` |
| About to tag a release | `vidhi-release-review` |
| Mid-upgrade checkpoint | `vidhi-release-review` (checkpoint mode) |
| Handing project to external reviewers | `vidhi-release-review` (pack is the handoff) |
| Forward-looking refactor planning | `vidhi-deepen` (not a review skill) |

## Finding schema

Both review skills use the same YAML finding format, so findings are interchangeable and can be reconciled across reviewers:

```yaml
- id: <stable-slug>
  severity: critical | high | medium | low
  category: correctness | contract | design | slop | performance
  title: <one line>
  location: <file:line | file | "project-wide">
  evidence: |
    <what's there now>
  why: |
    <why it matters>
  recommendation: |
    <one approach>
  confidence: high | medium | low
```

**Severity:**

| Level | Meaning | Blocks merge? |
|---|---|---|
| Critical | Breaks correctness, loses data, violates security | Yes |
| High | Will cause production problems or hurt the next person here | Yes |
| Medium | Real issue, bounded blast radius | No — fix soon |
| Low | Cleanup, naming, style | No — batch into follow-up |

## Installation

Add each skill directory to your Claude Code slash commands, or reference them via the skill index in `~/.claude/skill-index.md`.

## License

MIT — see [LICENSE](LICENSE). Originally by Matt Pocock; adapted by Josh Harper.
