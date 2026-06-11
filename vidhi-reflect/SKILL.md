---
name: vidhi-reflect
description: Mine the yojana corpus for recurring engineering lessons — cluster root causes, failed approaches, and decisions into themes, route each theme to exactly one enforcement tier (sutra rules, conventions, CLAUDE.md, playbook docs, remediation tasks, vidhi deltas), and report capture gaps. Use at release-review checkpoints (project scope) or periodically across all projects (global scope). vidhi-sutra-tend reconciles code against stated architecture; this reconciles practice against accumulated experience.
---

# Reflect (lesson extraction)

Turn closed yojana tasks into enforced lessons. Individual tasks record what
broke and why; nobody is assigned to notice that three projects hit the same
SQLite write hazard in three different costumes. This skill is that assignment.

This is an interactive skill. You mine, cluster, present a routed lesson plan,
and let the user shape it before writing. Lesson extraction is a taste
judgment — what generalizes vs. what was situational — so nothing is written
without the user seeing the routing first.

## Core idea: the lessons ledger

The deliverable is not prose insight — it is **routing of every accepted theme
to exactly one enforcement mechanism**, recorded in a ledger
(`~/soft/manas/docs/lessons/ledger.md`) the same way the enforcement ledger
records architectural claims. A lesson with no mechanism is a blog post; a
mechanism with no provenance is folklore. Every ledger row carries the yojana
task ids that evidence it.

| Tier | Mechanism | When |
|---|---|---|
| (a) | **Per-project sutra constraint** — rules.toml delta, tend discipline | graph-expressible: dependency seams, `confined_external`, cycles |
| (b) | **Sutra convention** — lifecycle to preferred/forbidden | code pattern FCA can detect |
| (c) | **`~/CLAUDE.md` lessons section** — hard budget, see below | agent-behavioral, applies across projects |
| (d) | **Project CLAUDE.md** | agent-behavioral, one project |
| (e) | **Playbook doc** — `~/soft/manas/docs/lessons/<theme>.md` | too big for any prompt; (c)/(d) entries point at it |
| (f) | **Remediation yojana tasks** | a specific codebase needs a specific fix |
| (g) | **Vidhi skill delta** | a process lesson — a skill step would have prevented the class |

Routing is exclusive. A theme that wants both a behavioral rule and code fixes
is two rows: the rule at (c)/(d), the fixes at (f).

## When to run

- **Project scope** — at vidhi-release-review, after the review itself: mine
  that project tree since its last reflect pass.
- **Global scope** — on demand or roughly quarterly: cross-project sweep
  looking for themes no single project can see.

Not per-task — close-out fields capture per-task learning (see
capture_discipline in manas-instructions); reflect aggregates them. Not
autonomous/scheduled — the routing judgments need a human until the taxonomy
has proven itself over several passes.

## Preconditions

- Yojana DB readable at `~/.yojana/yojana.db` (mine read-only via `sqlite3
  -readonly` — the MCP list shapes don't return close-out fields, and mining
  wants SQL anyway).
- Scope decided: a project slug (includes descendant subprojects) or global.
- `~/soft/manas/docs/lessons/ledger.md` — created on first run with a
  header recording the high-water mark per scope.

## Process

### 1. Establish the window

Read the ledger header for the scope's last-pass watermark (max `completed_at`
mined). First run: no watermark, mine everything, and say so — the first
global pass is the big one.

### 2. Mine

Query terminal tasks (`done`, `wontfix`) completed inside the window,
prioritizing lesson-bearing material:

```sql
-- the spine: bugs with mechanism recorded
SELECT p.slug || '/' || t.sequence_number, t.title, t.root_cause
FROM tasks t JOIN projects p ON p.id = t.project_id
WHERE t.status IN ('done','wontfix') AND t.completed_at > :watermark
  AND (t.category = 'bug' OR t.root_cause IS NOT NULL
       OR t.execution_record IS NOT NULL OR t.decisions != '[]'
       OR t.status = 'wontfix');
```

Read, per task: `root_cause` (the mechanism), `execution_record` (divergence
from plan, failed approaches), `decisions` (rationale + rejected
alternatives), wontfix rationales, and `task_conversations` comments for
closures that put the post-mortem there. Bug titles without root_cause are
weak evidence — usable to corroborate a theme, never to found one.

Global scope over a large window: fan out one subagent per project tree,
returning candidate themes with task-id evidence; cluster centrally. Project
scope: inline.

### 3. Cluster into candidate themes

Group along whichever axis the evidence suggests — technology ("SQLite FTS5
under concurrent writers"), failure mode ("stale long-lived connection view"),
process ("plan omitted migration step, found in execution"). A candidate
theme needs:

- **≥2 independent occurrences** (different tasks; for global themes, prefer
  different projects), or
- **1 severe occurrence** whose mechanism is clearly general, stated as such.

One-off mistakes are noise. Writing them down as global rules is how
CLAUDE.md becomes a junk drawer that erodes trust in every entry.

Check each candidate against existing ledger rows: a recurrence of an
extracted theme appends task ids to its row (and questions whether the chosen
mechanism is actually working — a tier-(c) rule that keeps recurring wants
promotion to (a)/(b)/(f), not a louder sentence).

### 4. Route

Assign each accepted theme a tier. Rules of the road:

- **Prefer the most mechanical tier that fits.** Enforced-by-guard beats
  detected-by-FCA beats remembered-by-agent. (c)/(d) is the fallback, not the
  default.
- **(a)/(b) deltas go through the owning repo's governance** — append to its
  rules.toml with `provenance = "reflect:<ledger-row>"`, severity per
  vidhi-sutra-adopt classification (existing violations → advisory). If the
  repo is governed, this is a mini-tend; do not regenerate anything.
- **Tier (c) is budgeted**: the `<engineering_lessons>` section of
  `~/CLAUDE.md` holds at most ~10 entries / ~40 lines. Adding an entry over
  budget requires evicting one — demote the evictee to a playbook (e) and
  leave a one-line pointer. Each entry is 1-3 sentences, imperative, with no
  war story; the story lives in the ledger row.
- **Tier (e) playbooks** are the deep version: the mechanism, the discipline,
  the evidence. Long-term this is vidya's lane; until vidya exists they live
  in manas docs.
- **Tier (f) tasks** are filed with status per triage_discipline and a
  `yojana:task` context_ref back to the evidencing tasks.
- **Tier (g)** names the skill and the step ("vidhi-plan premortem should ask
  about write-path concurrency for any embedded-DB design").

### 5. Capture-gap report

Count closures in the window that violated capture discipline: bugs closed
without root_cause, wontfix without rationale, plans that visibly diverged
with no execution_record. List the ones worth backfilling (severe or
theme-adjacent). This section is mandatory — when mining comes back thin,
the gap report is the pass's main product, and it feeds the discipline loop
rather than quietly underdelivering.

Present the backfill list as a **batch triage**: one line per item, the user
answers inline ("mechanism was X" / "superseded by Y" / "skip"), and you
write the backfills immediately. Human CLI closures are *expected* to arrive
bare — the human is not bound by agent close-out discipline. Never propose
blocking human closes.

But human memory decays faster than reflect cadence: the primary capture for
human closes is the close-time CLI shorthand (`--reason`, `-m`; yojana/40),
not this triage. Treat batch triage as the net for what still arrives bare,
expect "don't remember" on anything old, and record those as permanent gaps
without nagging — a skipped backfill twice is closed, not re-asked.

### 6. Present the plan

One table: theme · evidence (task ids) · tier · mechanism · draft text. Plus
the capture-gap list and any proposed evictions from the CLAUDE.md budget.
Ask the user to adjust routing before writing. Wait.

### 7. Write

- **Ledger** — append rows (theme, evidence ids, tier, mechanism, status,
  date); update the scope watermark in the header.
- **Tier artifacts** — rules.toml deltas (then `sutra_constraints list` +
  `violations` to verify they load), convention lifecycle calls, CLAUDE.md
  edits within budget, playbooks, remediation tasks, vidhi skill edits.
- Commit doc/rule changes per owning repo, one unit each:
  `Reflect pass (<scope>): <n> themes routed, <m> capture gaps`.

### 8. Hand off

Comment on the checkpoint task (or yojana/36-style tracking task for global
passes): themes routed, gaps found, next natural pass. If the same
graph-expressible theme has now recurred across ≥2 projects' rules.toml,
file a sutra task proposing global rules support — that's the evidence
threshold for building enforcement infrastructure, not before.

## Relationship to siblings

| Skill | Reads | Writes | Cadence |
|---|---|---|---|
| `vidhi-sutra-tend` | rules.toml + ledger vs. import graph | governance deltas | review checkpoints |
| `vidhi-release-review` | the release diff | findings, tasks | release |
| `vidhi-reflect` | yojana corpus vs. lessons ledger | routed lessons, capture-gap report | checkpoints (project) / quarterly (global) |

Tend catches drift between stated and actual architecture; reflect catches
repetition between projects' mistakes. Both produce deltas with provenance,
never regenerations.
