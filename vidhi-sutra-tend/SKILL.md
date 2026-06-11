---
name: vidhi-sutra-tend
description: Checkpoint maintenance for a sutra-governed repo — fire due deferred-constraint triggers, catch drift between stated architecture and the actual graph, constrain emergent structure, and triage conventions. Use at review checkpoints, after a track lands, or when the import graph has grown structure the existing rules don't cover. vidhi-sutra-seed births governance, vidhi-sutra-adopt retrofits it, this keeps it true.
---

# Sutra Tending (checkpoint maintenance)

Reconcile a governed codebase against its own stated architecture, and extend
governance to structure that didn't exist last time. The inputs — `.sutra/rules.toml`
and the enforcement ledger — are **authoritative and append-only**: this skill
produces deltas, never regenerations. If you find yourself proposing to rewrite
the rules file from the graph, you are running the wrong skill; the graph
includes whatever drift has already happened, and rules derived from it would
bless that drift.

This is an interactive skill. You diff, propose deltas, and let the user shape
them before writing.

## When to run

- A review checkpoint task fires (vidhi-review on a wave, vidhi-release-review)
  — especially the **first** one, when module interiors exist for the first time
- A track/PRD lands and its code has settled
- A deferred constraint's trigger event has occurred
- Sutra itself shipped features that affect how rules should be expressed

Not per-task — vidhi-review covers that. Not for ungoverned repos — that's
vidhi-sutra-adopt (existing code) or vidhi-sutra-seed (no code yet).

## Preconditions

- Sutra MCP available; workspace registered and freshly parsed (`sutra_status`).
- `.sutra/rules.toml` exists. If not, stop — seed or adopt first.
- An enforcement ledger (`docs/enforcement-ledger.md`) if the repo was seeded.
  Adopted repos may lack one; offer to backfill it from rules.toml provenance
  so every governed repo converges on the same maintenance loop.

## Process

### 1. Orient

Read `.sutra/rules.toml` and the ledger in full. Identify the checkpoint
context — the yojana task this pass runs under (review task, track-landing
task). Its id becomes the provenance stamp for everything this pass adds:
`checkpoint:<project>/<n>`. Reparse if stale.

### 2. Fire due triggers

Scan the deferred entries (ledger bucket b / commented `# TRIGGER:` blocks in
rules.toml). For each, has the trigger event occurred?

- **Fired** → bind it: uncomment, fix paths to the real layout, verify it
  binds (no `dead_constraint` warning), flip the ledger row to live.
- **Not fired** → leave it, but sanity-check the trigger still names a real
  future event (a task that was cancelled orphans its triggers).
- **Obsolete** (the structure it anticipated never materialized) → remove the
  block, record why in the ledger. Removal with rationale is honest; a
  permanently-deferred constraint is a lie in the ledger.

### 3. Health-check existing rules

```
sutra_constraints(workspace="<name>", action="list")
sutra_constraints(workspace="<name>", action="violations")
```

Two failure classes, opposite responses:

- **`dead_constraint` warnings** — a glob that used to bind went inert
  (directory renamed, crate restructured). This is silent unenforcement, the
  exact thing the ledger exists to prevent. Re-point the glob; never delete
  the rule just because it stopped matching.
- **Real violations** — drift that got past the guard (merged before rules,
  guard bypassed, advisory ignored). Route each: fix the code, or waive with
  rationale (`sutra_constraints action="waive"`). Silently downgrading
  severity to make a violation disappear is not an option you present.

### 4. Stated-vs-actual diff

The ledger is the stated architecture — far stronger input than the
README-archaeology vidhi-sutra-adopt has to work with. For each ledger claim,
check the graph (`sutra_deps`, `sutra_refs`, `sutra_grep`):

- **Confirmed** — note it; confirmations are what make the ledger trustworthy.
- **Contradicted** — drift. Present as a finding with the offending edges.
- **Unverifiable in the graph** — bucket (c)/(d) claims; spot-check the ones
  that are cheap to check (does the CLAUDE.md invariant still appear? did the
  promised tests land?) and flag broken ones.

### 5. Constrain emergent structure

The highest-value step, and the reason this skill exists. Seed-time
constraints guard the seams the PRD could name — typically crate-level. The
interiors grew since. Look for structure that exists now but has no rule:

```
sutra_map(workspace="<name>", limit=40)
sutra_components(workspace="<name>")
sutra_deps(workspace="<name>")
sutra_hotspots(workspace="<name>")
```

- **New layer boundaries** — handler/domain/infra splits inside a crate that
  the PRD never mentioned (route-class modules, middleware stacks, state)
- **Hub modules** — high fan-in files that aren't infrastructure: now there's
  real data, propose `max_fan_in` ~30% above current
- **Cycle-prone directories** — handler/adapter dirs where each unit should
  be independent: `no_cycles`
- **New external crates** — dependencies added since last pass that deserve
  confinement ("only the db module touches X")

Propose each as a new constraint with `provenance = "checkpoint:<task-id>"`
and a one-line rationale. Apply vidhi-sutra-adopt's classification discipline
(debt → advisory, clean → blocking) — unlike seed time, violations can exist
now, so severity is a real decision again.

### 6. Convention triage

Run vidhi-sutra-adopt §3h with whatever FCA has detected since last pass:
suppress tautologies, promote meaningful patterns to `preferred`, deprecate
anti-patterns, review pending promotion proposals. At the first checkpoint
this is the *initial* triage — FCA had nothing at seed time.

### 7. Present the delta plan

Organized as: triggers fired (bindings) · rule health (re-pointed globs,
violation routing) · drift findings · new constraints (emergent structure) ·
convention actions · ledger updates. Every item is an append or an explicit,
rationaled change — if the plan contains "regenerate" or an unexplained
removal, rewrite the plan. Ask the user to adjust before writing. Wait.

### 8. Write the deltas

- **rules.toml** — new constraints under a dated checkpoint section header;
  fired triggers uncommented in place; re-pointed globs edited in place with
  a comment noting the re-point and why.
- **Ledger** — flip deferred statuses, append rows for new constraints and
  drift findings, record obsoleted entries with rationale, stamp the pass
  (date + checkpoint task id) in the maintenance note.
- **Conventions** — `sutra_conventions` lifecycle calls as agreed.
- **CLAUDE.md** — only if a bucket-(c) invariant changed or a new
  non-expressible claim surfaced.

### 9. Verify and commit

Reparse, then `sutra_constraints` list + violations. Expect: all constraints
load, zero `dead_constraint` warnings (everything binds now — this repo has
code), and only violations that were explicitly waived or accepted as
advisory in step 7. Commit as one unit:

```
Tend sutra governance at <checkpoint>

<n> triggers fired, <m> constraints added for emergent structure,
<k> conventions triaged. Drift: <one line or "none">.
```

### 10. Hand off

Comment on the checkpoint task: what was bound, added, found. If drift
findings need code changes beyond this pass, file them as yojana tasks (with
user confirmation). Note the next natural tend moment if one is visible
(next review checkpoint, next track landing).

## Relationship to siblings

| Skill | Moment | Input | Output |
|---|---|---|---|
| `vidhi-sutra-seed` | birth — PRD, no code | PRD claims | rules.toml + ledger, total routing |
| `vidhi-sutra-adopt` | adoption — code, no governance | import graph + doc archaeology | rules.toml from discovered architecture |
| `vidhi-sutra-tend` | checkpoint — code and governance | ledger + rules.toml vs actual graph | deltas: bindings, drift findings, new constraints, convention triage |

Tend is for what couldn't be known at seed/adopt time — not for what was
skipped. If seed under-constrained on the theory that tend would catch it,
that's a seed-discipline bug; the guard was off exactly when edits were
cheapest to shape.
