---
name: vidhi-reflect
description: Mine the yojana corpus for recurring engineering lessons — cluster root causes, failed approaches, and decisions into themes, route each theme to exactly one enforcement tier (sutra rules, conventions, lessons store, project CLAUDE.md, playbook docs, remediation tasks, vidhi deltas), prune/flag/dedup the lessons store, and report capture gaps. Use at release-review checkpoints (project scope) or periodically across all projects (global scope). vidhi-sutra-tend reconciles code against stated architecture; this reconciles practice against accumulated experience.
---

# Reflect (lesson extraction)

Turn closed yojana tasks into enforced lessons. Individual tasks record what
broke and why; nobody is assigned to notice that three projects hit the same
SQLite write hazard in three different costumes. This skill is that assignment.

This is an interactive skill. You mine, cluster, present a routed lesson plan,
and let the user shape it before writing. Lesson extraction is a taste
judgment — what generalizes vs. what was situational — so nothing is written
without the user seeing the routing first.

## Core idea: the lessons ledger + lessons store

Two complementary records:

- **Ledger** (`~/soft/manas/docs/lessons/ledger.md`) — the routing table of
  record. Every accepted theme maps to exactly one enforcement mechanism;
  every row carries the yojana task IDs that evidence it. A lesson with no
  mechanism is a blog post; a mechanism with no provenance is folklore.
- **Lessons store** (`~/.sutra/lessons.db` via `sutra_remember`) — the
  contextual surfacing layer. Tier (c) lessons live here, anchored to
  symbols/files/imports so sutra surfaces them to agents working in the
  relevant code. Quality controlled by a confidence lifecycle: born
  unverified, gain confidence through citation, decay/archive if uncited.

The ledger records *what was routed where and why*. The lessons store is *the
enforcement mechanism* for tier (c). Both are maintained by this skill.

| Tier | Mechanism | When |
|---|---|---|
| (a) | **Per-project sutra constraint** — rules.toml delta, tend discipline | graph-expressible (dependency seams, `confined_external`, cycles), or pattern-expressible via `forbidden_pattern` from `vidhi/language-rules/<lang>.toml` — the only tier that fires at edit time |
| (b) | **Sutra convention** — lifecycle to preferred/forbidden | code pattern FCA can detect |
| (c) | **Lessons store** via `sutra_remember` — contextually surfaced by sutra tools | agent-behavioral, applies across projects |
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
- Lessons store reachable at `~/.sutra/lessons.db` (sutra MCP must be
  running for `sutra_remember` calls).
- `~/soft/manas/docs/lessons/ledger.md` — the routing table of record.

## Process

### 1. Establish the window

Read the watermark from the lessons store metadata table:

```sql
-- Bootstrap (first run only):
CREATE TABLE IF NOT EXISTS metadata (
    key   TEXT NOT NULL PRIMARY KEY,
    value TEXT NOT NULL
);

-- Read watermark:
SELECT value FROM metadata
WHERE key = 'reflect_watermark:<scope>';
```

Where `<scope>` is the project slug or `global`. The value is the max
`completed_at` timestamp (integer ms, matching yojana's format) from the
previous pass. First run: no watermark row, mine everything, and say so —
the first global pass is the big one.

Project scope inherits the global floor: use
`max(reflect_watermark:<slug>, reflect_watermark:global)` as the window
start. A global pass already mined everything below the global mark, so a
project pass with no per-project watermark must not re-mine that project's
full history — it would redo covered work and re-present backfill items the
user already triaged. (Global scope reads only its own key; per-project
watermarks don't advance it.)

The watermark MUST be the max `completed_at` **of the rows actually mined**,
never the pass's own wall-clock time. A wall-clock watermark skips any task
that closes while the pass is running (the chitta/43 bug class — its
reflect_status used run-completion time and could skip rows that arrived
mid-run). Max-of-mined is safe: anything closing during the pass gets a
`completed_at` above it and is caught next time.

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

Also check against existing lessons in the store — a candidate matching an
existing lesson is evidence for citation, not a new lesson:

```
sutra_remember cite="<lesson_id>" source_tasks=["<task_id>"]
```

### 4. Lessons store maintenance

Before routing new themes, maintain the existing store. These four passes run
every reflect invocation.

#### 4.1 Prune decayed lessons

`LessonsDb::archive_decayed()` runs automatically on DB open with a 90-day
window, silently archiving the obvious cases (no citations, no surfacings,
past window). This pass reviews the borderline cases with human judgment.

Find unverified lessons that have been surfaced but never cited — they reached
agents but never proved useful enough to reference in a close-out:

```sql
SELECT l.id, l.text, l.created_at, l.last_surfaced, l.confidence,
       l.project_origin
FROM lessons l
WHERE l.archived = 0 AND l.verified = 0
  AND l.last_surfaced IS NOT NULL
  AND (l.last_cited IS NULL OR l.last_cited < datetime('now', '-90 days'))
  AND l.created_at < datetime('now', '-90 days')
ORDER BY l.created_at;
```

Present each to the user: "Surfaced but never cited in 90+ days — keep or
archive?" The user answers inline. Archive decisions are immediate:

```sql
UPDATE lessons SET archived = 1 WHERE id = '<id>';
```

Report: "Pruned N lessons (M auto-archived by decay, K archived after
review, J kept by user)."

#### 4.2 Flag stale verified lessons

Check verified lessons for anchor code drift. The staleness machinery
exists in sutra (`LessonsDb::check_staleness` + `anchor_verification` table
with content hashes snapshotted at verification time).

```sql
SELECT l.id, l.text, l.verified_at,
       a.kind, a.value, av.content_hash
FROM lessons l
JOIN anchor_verification av ON av.lesson_id = l.id
JOIN anchors a ON a.id = av.anchor_id
WHERE l.verified = 1 AND l.archived = 0;
```

For each verified lesson, compare the stored `content_hash` against the
current hash of the anchored symbol/file (resolve via the workspace index).
If changed, present to user with three choices:

- **Still valid** → re-cite the lesson to refresh verification hashes:
  `sutra_remember cite="<lesson_id>" workspace="<path>"`
- **Needs update** → edit lesson text, re-store as new lesson with same
  anchors, archive the stale one
- **Archive** → `UPDATE lessons SET archived = 1 WHERE id = '<id>';`

This is the human-in-the-loop complement to sutra's automatic `[stale]`
flagging during surfacing. Surfacing flags stale lessons for agents to treat
with caution; reflect resolves them.

#### 4.3 Deduplicate overlapping lessons

Find lessons sharing anchors that may say the same thing:

```sql
SELECT a1.lesson_id AS id1, a2.lesson_id AS id2,
       COUNT(*) AS shared_anchors
FROM anchors a1
JOIN anchors a2 ON a1.kind = a2.kind AND a1.value = a2.value
     AND a1.lesson_id < a2.lesson_id
JOIN lessons l1 ON l1.id = a1.lesson_id AND l1.archived = 0
JOIN lessons l2 ON l2.id = a2.lesson_id AND l2.archived = 0
WHERE a1.kind IN ('symbol', 'file')
GROUP BY a1.lesson_id, a2.lesson_id
HAVING shared_anchors >= 1
ORDER BY shared_anchors DESC;
```

The `WHERE a1.kind IN ('symbol', 'file')` clause is load-bearing. `directory`
and `import_pattern` anchors are deliberately coarse — two lessons about
completely different concerns routinely share `src/db` or `sqlx::*` — so
counting them turns "lives near" into a false duplication signal. Measured on
the 2026-07-22 store: 7 genuine candidate pairs with the clause, 17 without.

For each pair with shared anchors, read both lesson texts and judge: are they
saying the same thing in different words? Present merge candidates:

- "Lessons `<id1>` and `<id2>` share N anchors. Text comparison: [show both].
  Merge? Which text to keep?"

Merge operation (when user approves):
1. Keep the higher-confidence lesson
2. Add the other's unique anchors to the keeper
3. Move the other's citations to the keeper
4. Archive the duplicate

Automated merge is too risky — two lessons anchored to the same file may be
about completely different concerns. This pass always presents and waits.

#### 4.4 Audit write-time reachability

Every sutra surfacing point is a read operation, so a lesson anchored only to
symbols and files can never reach an agent writing new code. This pass finds
lessons stranded that way:

```sql
SELECT l.id, substr(l.text, 1, 120) AS txt, l.project_origin,
       (SELECT group_concat(a.kind || ':' || a.value, '  ')
          FROM anchors a WHERE a.lesson_id = l.id) AS anchors
FROM lessons l
WHERE l.archived = 0
  AND NOT EXISTS (SELECT 1 FROM anchors a
                   WHERE a.lesson_id = l.id
                     AND a.kind IN ('directory', 'import_pattern'))
ORDER BY l.created_at;
```

A hit is not automatically a defect. Ask per lesson: *would this matter to
someone writing a file that doesn't exist yet?*

- **No** — the lesson is genuinely about existing code ("when modifying
  `Ephemeris::apply_sidereal`, watch the sidereal speed path"). Read-time
  anchoring is correct. Leave it.
- **Yes** — add a `directory` or `import_pattern` anchor per the tier (c)
  guidance in step 5.

Derive directory anchors from the lesson's existing `file` anchors, skipping
any parent with fewer than two path components — a bare `src`/`lib` anchor
fires on every file in the repo and buys noise, not reach.

Report: "Reachability: N lessons read-time-only, K re-anchored, J confirmed
read-time by design."

### 5. Route

Assign each accepted theme a tier. Rules of the road:

- **Prefer the most mechanical tier that fits.** Enforced-by-guard beats
  detected-by-FCA beats remembered-by-agent. (c) is the fallback, not the
  default.
- **Prefer write-time delivery over read-time recall.** A lesson only helps if
  it surfaces while the work is happening. Every sutra surfacing point
  (`sutra_read`, `sutra_impact`, `sutra_orient`) is a *read* operation, so a
  (c) lesson anchored only to symbols is invisible to an agent writing new
  code. Before settling on (c), ask whether the theme is pattern-expressible —
  if it is, it belongs at (a) as a `forbidden_pattern`, which fires in the
  editor and needs no agent to remember anything.
- **(a)/(b) deltas go through the owning repo's governance** — append to its
  rules.toml with `provenance = "reflect:<ledger-row>"`, severity per
  vidhi-sutra-adopt classification (existing violations → advisory). If the
  repo is governed, this is a mini-tend; do not regenerate anything.
- **Tier (c) goes to the lessons store** via `sutra_remember`:
  - `text`: imperative, 1-3 sentences, no war story — the story lives in the
    ledger row
  - `location_anchors`: symbols or files from the evidence tasks' `files`
    fields or root_cause descriptions — then **check write-time reachability
    before you settle**. Ask: *does this lesson apply to a file that doesn't
    exist yet?* If yes, symbol and file anchors cannot reach it and the lesson
    also needs one of:
    - a `directory` anchor — for location-scoped lessons ("everything under
      `migrations/` must be idempotent"). Scope it to a real subdirectory;
      a bare `src`/`lib` fires repo-wide and is noise, not reach.
    - an `import_pattern` anchor — for technology-scoped lessons that follow
      the dependency rather than the location ("`sqlx::query_as` panics on an
      empty result set"). These travel to new code the symbol anchors never see.
  - `source_tasks`: the yojana task IDs from the evidence column
  - `project_origin`: null for cross-project themes, project slug for
    single-project themes
  - `categories`: technology/concern tags (e.g., `["sqlite", "concurrency"]`)
  - `workspace`: **always pass it.** Sutra auto-adds import-pattern anchors,
    directory anchors, and language categories from the workspace graph.
    Omitting it is the main reason stored lessons end up read-time-only.
  - No cardinality cap — sutra's contextual surfacing replaces the broadcast.
    Anchor specificity and category filtering naturally limit what surfaces.
- **Tier (e) playbooks** are the deep version: the mechanism, the discipline,
  the evidence. Long-term this is vidya's lane; until vidya exists they live
  in manas docs.
- **Tier (f) tasks** are filed with status per triage_discipline and a
  `yojana:task` context_ref back to the evidencing tasks.
- **Tier (g)** names the skill and the step ("vidhi-plan premortem should ask
  about write-path concurrency for any embedded-DB design").

### 6. Capture-gap report

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

### 7. Present the plan

One table: theme · evidence (task ids) · tier · mechanism · draft text. Plus
the capture-gap list, the maintenance report (pruned/stale/deduped/
reachability), and any proposed tier (c) `sutra_remember` calls with their
anchors and categories.
Ask the user to adjust routing before writing. Wait.

### 8. Write

- **Ledger** — append rows (theme, evidence ids, tier, mechanism, status,
  date). For tier (c), the mechanism column records `sutra_remember lesson
  <lesson_id>` after the lesson is stored. Update the scope watermark in
  the metadata table:
  ```sql
  INSERT OR REPLACE INTO metadata (key, value)
  VALUES ('reflect_watermark:<scope>', '<max_completed_at>');
  ```
- **Tier (c) lessons** — store via `sutra_remember`:
  ```
  sutra_remember(
    text="<lesson text>",
    location_anchors=[{kind: "symbol", value: "<name>"}, ...],
    source_tasks=["<task/N>", ...],
    project_origin="<slug or null>",
    categories=["<tag>", ...],
    workspace="<path>"
  )
  ```
  Record the returned `lesson_id` in the ledger row's mechanism column.
- **Other tier artifacts** — rules.toml deltas (then `sutra_constraints list`
  + `violations` to verify they load), convention lifecycle calls, project
  CLAUDE.md edits, playbooks, remediation tasks, vidhi skill edits. These
  are unchanged from prior behavior.
- **Maintenance results** — pruning archives, staleness resolutions, dedup
  merges already written during §4.
- Commit doc/rule changes per owning repo, one unit each:
  `Reflect pass (<scope>): <n> themes routed, <m> capture gaps, <k> lessons maintained`.

### 9. Hand off

Comment on the checkpoint task (or yojana/36-style tracking task for global
passes): themes routed, gaps found, lessons maintained (pruned/flagged/deduped/
re-anchored), next natural pass. If the same graph-expressible theme has now
recurred across ≥2 projects' rules.toml, file a sutra task proposing global
rules support —
that's the evidence threshold for building enforcement infrastructure, not
before.

## Relationship to siblings

| Skill | Reads | Writes | Cadence |
|---|---|---|---|
| `vidhi-sutra-tend` | rules.toml + ledger vs. import graph | governance deltas | review checkpoints |
| `vidhi-release-review` | the release diff | findings, tasks | release |
| `vidhi-reflect` | yojana corpus vs. lessons ledger + lessons store | routed lessons, store maintenance, capture-gap report | checkpoints (project) / quarterly (global) |

Tend catches drift between stated and actual architecture; reflect catches
repetition between projects' mistakes. Both produce deltas with provenance,
never regenerations.
