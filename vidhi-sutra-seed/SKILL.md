---
name: vidhi-sutra-seed
description: Seed sutra governance for a codebase that doesn't exist yet — derive .sutra/rules.toml, an enforcement ledger, and aliases.toml from a PRD instead of an import graph. Use at project birth, after a PRD exists but before any code is scaffolded. vidhi-sutra is the adoption pass for existing code; this is the birth pass.
---

# Sutra Seeding (greenfield)

Generate `.sutra/rules.toml`, a seed `aliases.toml`, and an **enforcement ledger**
for a codebase that doesn't exist yet, from its PRD. Where vidhi-sutra discovers
architecture from an import graph, this skill extracts it from stated intent —
and because there is no code to contradict the PRD, every claim is enforceable
from the first commit.

This is an interactive skill. You extract, classify, present the routing plan,
and let the user shape it before writing.

## Core idea: the enforcement ledger

The deliverable is not a rules file — it is **total routing of every
architectural claim in the PRD**. Each boundary, seam, or isolation statement
is routed to exactly one enforcement mechanism:

| Bucket | Mechanism | When |
|---|---|---|
| (a) | **Live sutra constraint** — written to rules.toml now | the PRD fixes the paths (workspace crate names, fixed module dirs) |
| (b) | **Deferred sutra constraint** — commented-out block with a binding trigger | needs code to exist first (`no_cycles` scopes, `max_fan_in` thresholds) |
| (c) | **Not expressible in sutra** — written into the consumer's CLAUDE.md or review checklist | runtime behavior, DB/RLS policy, process discipline |
| (d) | **Test-asserted** — the PRD's testing decisions, pointed at their implementing task | behavior the test suite will pin |

The ledger (`docs/enforcement-ledger.md` in the consumer repo) records every
claim with its quote, bucket, mechanism, and status. Its purpose is to make
silent non-enforcement impossible: a claim with no row is a routing bug.

Routing is exclusive — one claim, one mechanism. If a claim appears both as a
design statement and in the PRD's testing decisions, route it to (d); the test
is the stronger enforcement.

## Preconditions

- Sutra MCP server available (check for `sutra_status`). If not, stop and tell the user.
- A PRD exists — a yojana task (preferred, for provenance) or a doc path.
- A consumer directory identified (typically empty or near-empty).

## Process

### 1. Locate inputs

- The PRD: yojana task id (e.g. `project/1`) or doc path. Read it in full.
- The consumer root (infer from cwd or ask).
- If the PRD has been decomposed, identify the **scaffold task** — the task
  that will create the workspace skeleton. The seeded rules become its
  acceptance criteria (step 8).

### 2. Extract claims with quotes

Sweep the PRD for architectural claims. Taxonomy (vidhi-sutra step 2, adapted
for greenfield — deprecations rarely apply, external-dependency claims dominate):

- **Layer/crate boundaries** — "server is the only binary", "report knows nothing of Axum"
- **Single-point-of-contact** — "the only place X appears", "X is configured by a single env var"
- **Isolation/purity** — "pure function from inputs to bytes", "this is the extraction seam"
- **Forbidden dependencies** — license boundaries ("nothing links Y"), banned crates
- **Runtime behavior** — "degrade when X is down", "idempotent webhook processing"
- **Data/security policy** — RLS rules, schema visibility, role scoping, secrets handling
- **Process/API discipline** — versioning rules, migration ownership, host preferences
- **Testing decisions** — the PRD's own test plan, including manual-verification items

Record each claim verbatim (short quote) with its PRD section. Every claim
gets a ledger row — including ones you'll route to (c) or (d). Skill-standard
seeds you add that the PRD doesn't state (e.g. `no_cycles` on a handler dir)
get rows too, marked `source: skill default`.

### 3. Route and draft

Classify every claim into a bucket. Rules of the road:

**Timing rule.** A constraint binds to globs only when the PRD fixes the
paths. A glob matching zero files enforces nothing — speculative globs are
constraint theater. Workspace crate names fixed by the PRD → bucket (a).
Module layouts the scaffold will invent → bucket (b), with a trigger.

**Severity.** Greenfield default is `blocking` for everything. Advisory
severity exists to grandfather existing debt; a greenfield has none.

**Deferred constraints** are written into rules.toml as commented-out
`[[constraint]]` blocks under a `# TRIGGER:` comment naming the binding event
("at scaffold (task N): fix scope path and uncomment", "at first review
checkpoint"). Binding is then uncomment-and-verify, not re-derivation.

**Crate-to-crate seams are `forbidden_dep`.** Cross-crate import resolution
landed in sutra (needs-designing/15): sibling-crate imports resolve to real
edges. Never express an in-workspace seam with the external kinds — a
`forbidden_external`/`confined_external` whose `crates` names a workspace
member is a hard error once the workspace manifest exists.

**`max_fan_in` is always deferred** — not for tooling reasons, but because
thresholds need real fan-in data. Bucket (b), trigger at first review
checkpoint.

**Prefer `confined_external` over scattered forbids.** "Report knows nothing
of Axum" plus "the only binary is server" collapses to `confined_external
crates=["axum"] allowed_in=["server/**"]` — one rule, stronger than the sum of
per-crate forbids, and it covers crates the PRD didn't think to mention.

### 4. Present the routing plan

Before writing anything, show the user the full ledger as a table — claim,
quote, bucket, mechanism, trigger — plus the drafted constraint list (kind,
globs, name) and the bucket-(c) text destined for CLAUDE.md.

Ask: "Want to adjust any routing before I write?" They may rebucket claims,
drop or add constraints, change globs, or edit the CLAUDE.md items. Wait for
their response.

### 5. Write the artifacts

In the consumer root:

- **`.sutra/rules.toml`** — live constraints grouped by concern with section
  headers, each with `name`, explicit `severity`, and `provenance` pointing at
  the PRD (`yojana:<project>/<n> §<section>`). Deferred blocks commented out
  under their `# TRIGGER:` lines. No `[conventions]` suppressions — FCA has no
  data yet; convention triage belongs to the first review checkpoint.
- **`.sutra/aliases.toml`** — `[component]` entries from PRD vocabulary
  (crates/components only; symbol aliases come with real code).
- **`docs/enforcement-ledger.md`** — the routing table of record, with a
  status column (`live` / `deferred (trigger)` / `routed → CLAUDE.md` /
  `test (task N)` / `manual (task N)`) and a maintenance note: new track PRDs
  append rows; rebinding events check rows off.
- **`CLAUDE.md`** — bucket (c) items written as an invariants section, grouped
  (runtime, data/security, API discipline, process). Actually write them —
  "routed" means routed, not noted for later. If a CLAUDE.md exists, append
  the section.
- **`.gitignore`** — the sutra block (track config, ignore state):

  ```
  .sutra/*/
  .sutra/acks/
  !.sutra/rules.toml
  !.sutra/aliases.toml
  !.sutra/owners.toml
  ```

### 6. git init

If the consumer is not yet a git repository, `git init` and make the seed
artifacts the first commit — "guard on from first commit" is literal: the repo
is born governed. If a repo exists, commit the artifacts as a unit.

```
Seed sutra governance from PRD (<yojana id>)

N live constraints, M deferred; enforcement ledger routes K claims.
```

### 7. Register and verify

Register the workspace (`sutra_add_root`) and check `sutra_status`. Then:

```
sutra_constraints(workspace="<name>", action="list")
sutra_constraints(workspace="<name>", action="violations")
```

What this verifies at seed time: rules.toml **parses**, all N constraints
**load**, and there are 0 real violations. Expect exactly N `dead_constraint`
*informational* warnings — with no source files, every glob is inert, and
sutra says so explicitly. That's correct at seed time; record it in the
ledger. The scaffold task's acceptance criterion is precisely that those
warnings disappear (every glob binds), which is why step 8 exists. If sutra
can't register an empty workspace, record that and defer all verification to
scaffold.

### 8. Hand off to the scaffold task

Append machine-checkable acceptance criteria to the scaffold task (with a
`doc:path` context_ref to the ledger):

- `sutra_constraints list` shows all N constraints loaded, 0 violations
- every live constraint's glob binds ≥1 real file/crate after scaffold
- deferred blocks whose trigger is "at scaffold" are uncommented with real
  paths and verified
- sutra-guard wired as a PreToolUse hook in the repo's `.claude/settings.json`

Comment on the PRD task: seeding done, pointer to the ledger.

### 9. Lifecycle — keep with it

Record these in the ledger's maintenance note (and tell the user):

- **Guard from first commit** — blocking rules are live the moment code exists.
- **Per-task review** — `sutra_review` runs in vidhi-review per task; the
  ledger's (c)-bucket items are review-checklist material.
- **First review checkpoint** — run convention triage (vidhi-sutra step 3h)
  there, when FCA has real data; also the trigger for fan-in guardrails.
  vidhi-sutra is the adoption pass; it takes over from here.
- **New track PRDs** — re-run this skill additively: append constraints with
  their own provenance, append ledger rows. Never regenerate from scratch.

## Reference: constraint kinds for greenfield

The workhorses at seed time are the external kinds — they're path-glob ×
crate-name, so PRD-fixed crate names are enough to bind:

| Kind | Fields | Use |
|---|---|---|
| `forbidden_external` | `crates` (name globs), optional `from` (path glob, default `**`), `include_dev` | License boundaries, banned crates |
| `confined_external` | `crates`, `allowed_in` (path globs; `[]` = banned everywhere), `include_dev` | Single-point-of-contact ("protos only in quiver-client", "axum only in server") |
| `forbidden_dep` | `from`, `to` (path globs) | Crate-to-crate seams ("report must not depend on server") — never the external kinds for these; naming a workspace member there is a hard error |

Crate-name globs allowed; hyphens/underscores equivalent. The external kinds
check use-statements **and** Cargo.toml `[dependencies]`; dev-deps exempt
unless `include_dev = true`.

Defer until structure exists: `no_cycles` (scope is an invented module path),
`boundary` (components don't exist yet), `max_fan_in` (needs data). Full
table: vidhi-sutra § Reference.
