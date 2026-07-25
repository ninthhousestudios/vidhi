---
name: vidhi-sutra-adopt
description: Adopt an existing codebase into sutra — register the workspace, discover architecture from the import graph, and propose a .sutra/rules.toml with constraints and convention management. Use when bringing a codebase with existing code under sutra governance for the first time, or when resetting a project's constraint rules from scratch.
---

# Sutra Adoption

Generate a `.sutra/rules.toml` for a codebase by discovering its architecture, analyzing dependency patterns, and deriving constraints from what's actually there. The output is a reviewed, committed rules file — not a template.

This is an interactive skill. You explore, analyze, present findings, and let the user shape the rules before writing.

Siblings: for a codebase that doesn't exist yet (PRD but no code), use **vidhi-sutra-seed** — it derives rules from stated intent and hands binding verification to the scaffold task. For a repo that is already governed, use **vidhi-sutra-tend** — checkpoint maintenance that appends deltas, never regenerates. Seed births governance, this skill retrofits it, tend keeps it true.

## Preconditions

- Sutra MCP server must be available (check for `sutra_status` tool). If not, stop and tell the user.
- The codebase must be in a language sutra supports (Rust, Dart). If unsure, check with `sutra_help`.

## Entry

The user provides a path to the workspace root, or you infer it from the current working directory. If the workspace has a `Cargo.toml` or `pubspec.yaml`, auto-detect the language. Otherwise ask.

## Process

### 1. Register and parse

Check if the workspace is already registered:

```
sutra_status(path="/absolute/path/to/workspace")
```

If `status` is not `ready` or `is_stale` is true, trigger a parse:

```
sutra_parse(workspace="<name>")
```

Wait for parse to complete (re-check status). Report the file count, symbol count, and any parse errors.

### 2. Map the architecture

Run these in parallel to build a picture of the codebase:

```
sutra_map(workspace="<name>", limit=40)
sutra_components(workspace="<name>")
sutra_deps(workspace="<name>")
sutra_conventions(workspace="<name>", action="list")
```

Also explore the repo for architecture docs and extract **stated architectural intent**:

- Look for `docs/adr/` or similar ADR directories
- Read `CONTEXT.md`, `CLAUDE.md`, `README.md` for stated architectural intent
- Check for existing `.sutra/rules.toml` — if present, read it as a starting point
- Look for migration docs, punchlists, or design docs in `docs/`, `claude/arch/`, etc.

Extract concrete claims about intended structure. You're looking for statements like:

- **Layer boundaries** — "GUI must not call SWE directly", "display code must not do ephemeris", "options → swe → core → calc"
- **Forbidden dependencies** — "X should not import Y", "route through Z instead of calling W directly"
- **Intended deprecations** — "X is deprecated, use Y instead", "will be deleted in Issue N"
- **Isolation requirements** — "this module must stay pure", "these subsystems are independent"

Record each claim with its source (file path + relevant quote). These become the **stated architecture** — the intended design that the actual graph may or may not reflect. Even stale docs are valuable here: if a README says "A must not import B" but the graph shows A → B, that's either debt to surface or doc drift to flag. Both are useful findings for the human to resolve in step 4.

### 3. Analyze and classify

From the data gathered, build a mental model of the codebase organized into these categories. Present your findings to the user as a structured summary before proposing rules.

#### 3a. Layer structure

Identify the dependency layers from the import graph. Common patterns:

- **Infrastructure** — error types, config, utilities (high fan-in, imported everywhere)
- **Data layer** — DB access, persistence, models
- **Domain/engine** — core business logic, algorithms
- **Presentation** — CLI handlers, API endpoints, MCP tools, UI
- **Orchestration** — pipelines, coordinators that wire layers together
- **Tests** — integration and unit test files

Show the user which files belong to which layer and the dependency direction. Flag any **upward dependencies** (lower layer importing higher layer).

#### 3b. Stated vs. actual architecture

Compare the stated architectural intent (from step 2) against the actual import graph. For each stated claim, check whether the graph confirms or contradicts it.

Present mismatches in a table:

| Stated intent | Source | Actual graph | Classification |
|---|---|---|---|
| "display code must not do ephemeris" | `docs/migration.md` | `apps/core_gui_qt.py → swe` (12 edges) | Debt — propose advisory constraint |
| "options → swe → core → calc" | `CLAUDE.md` | All edges respect this | Confirmed — propose blocking constraints |
| "X is deprecated, use Y" | `README.md` | 4 files still import X | Debt — propose advisory constraint |

Classify each mismatch:

- **Confirmed** — graph matches stated intent → propose as blocking constraint with `provenance` pointing to the source doc
- **Architectural debt** — graph violates stated intent, but the violation is widespread or has pragmatic reasons → propose as advisory constraint, note the gap
- **Doc drift** — stated intent seems outdated (e.g., doc describes a module that no longer exists) → flag for the human but don't propose a constraint

If no architecture docs exist, skip this sub-step and note the gap: "No stated architecture found — constraints below are derived purely from the import graph."

#### 3c. Naming heuristic check

As a fallback that runs regardless of whether architecture docs exist, apply naming heuristics to flag suspicious cross-layer edges:

**Presentation-layer directories** (heuristic): `apps/`, `gui/`, `views/`, `widgets/`, `ui/`, `pages/`, `screens/`, `handlers/`, `routes/`

**Infrastructure-layer modules** (heuristic): names containing `swe`, `db`, `sql`, `ffi`, `binding`, `raw`, `wire`, `proto`

Flag any edge where a presentation-layer file imports an infrastructure-layer module directly — these are candidates for "should this go through an intermediary?" Present them to the user as questions, not assertions. False positives are fine here; the human resolves them in step 4.

This check is supplementary. It catches obvious layer violations even when docs don't exist or don't mention them. It does NOT replace the stated-vs-actual comparison — naming is a weak signal; docs are strong signal.

#### 3d. Cross-layer couplings

From the dependency graph (and informed by 3b/3c), identify imports that cross layer boundaries in the wrong direction. Classify each:

- **Architectural debt** — exists for pragmatic reasons, should be advisory constraints to document and create pressure
- **Clean violations** — clearly wrong, should be blocking constraints
- **Legitimate** — the layers don't apply here (e.g., pipeline/orchestrator importing everything is expected)

#### 3e. Isolation boundaries

Look for modules that must stay lightweight or pure. Common patterns:

- **Guard/hook binaries** — real-time tools that can't afford heavy deps
- **Worker threads** — pure computation communicating via channels
- **Shared libraries/crates** — reusable code that shouldn't depend on application-specific modules
- **Plugin/adapter interfaces** — should depend on traits, not implementations

#### 3f. Hub modules

From `sutra_map`, identify files with high fan-in that aren't infrastructure. These are candidates for `max_fan_in` constraints — not to prevent legitimate use but to detect when a module is accumulating too many responsibilities.

Reasonable thresholds:
- Infrastructure (error, config, utilities): no limit needed
- Domain modules: 10–20 depending on codebase size
- Avoid setting limits so tight they trigger on current state — set them ~30% above current fan-in as guardrails

#### 3g. Cycle candidates

Identify directory subtrees where cycles would be architecturally harmful. Good candidates for `no_cycles`:

- Tool/handler directories (each handler should be independent)
- Plugin/adapter directories
- Core domain module directories

Don't add `no_cycles` to the entire `src/` — it's too broad and legitimate reference cycles exist in Rust (e.g., `mod.rs` re-exporting from submodules).

#### 3h. Pattern discipline

Read `vidhi/language-rules/{language}.toml` for the project's language(s).
For each catalog rule, run the query against the codebase (or estimate from
grep as a proxy) to understand the current state:

- **Match count** — how many existing matches would this rule surface?
- **Distribution** — are matches concentrated (one legacy module) or pervasive?

Present each rule with its `description`, current match count, and
`false_positives` guidance. Let the user decide:

- **Adopt as advisory** — document the discipline, surface in review
- **Adopt as blocking** — enforce at edit time (only reasonable if match
  count is low or all existing matches will be waived)
- **Skip** — not relevant to this codebase
- **Defer** — adopt at a later checkpoint after cleanup

For adopted rules, the scope comes from `scope_hint` adjusted to match the
project layout. Blocking patterns with existing matches need waivers planned
(same as blocking dep constraints with existing violations in step 7).

#### 3i. Convention triage

Review the auto-detected conventions. Classify them:

- **Tautological** — "symbols in directory X are in directory X" (component-scoped conventions where the consequent is just `in:<directory>`). Add to suppress list.
- **Structural truths** — "test functions are functions" (`in:tests → kind:function`). Suppress.
- **Meaningful patterns** — "public functions return Result", "methods take &self". Leave active; candidates for promotion to preferred.
- **Potentially wrong** — patterns that exist due to coincidence, not intent. Leave descriptive, monitor.

Also check for pending promotion proposals and flag ones worth accepting.

### 4. Present the constraint plan

Before writing any file, present the full plan to the user organized by category:

**Stated-vs-actual mismatches** (from 3b) — if any stated architectural intent was contradicted by the graph, present the mismatch table first. For each, show the proposed constraint (advisory or blocking) and whether it's debt or doc drift. This is the highest-signal section — these constraints come from human-authored intent, not just graph analysis.

**Naming heuristic flags** (from 3c) — if any suspicious cross-layer edges were found, present them as questions: "Should `apps/foo.py` import `swe` directly, or should this go through an intermediary?" Let the human classify each as intentional, debt, or false positive.

**Blocking constraints** — the rules the guard will enforce in real time.
Show each with: kind, from/to (or scope), name, and one-line rationale. Include `provenance` for any constraint derived from a stated architecture doc.

**Advisory constraints** — known debt documented as rules.
Show each with the current violation and why it's advisory not blocking.

**Cycle prevention** — which scopes and why.

**Fan-in limits** — which modules, what threshold, current fan-in value.

**Convention suppressions** — which convention IDs and what they say.

Ask the user: "Want to adjust any of these before I write the file?"

Wait for their response. They may:
- Accept as-is
- Remove constraints they disagree with
- Add constraints you didn't propose
- Change severities
- Adjust thresholds

### 5. Write rules.toml

Write `.sutra/rules.toml` using the agreed constraints. Follow this format:

```toml
# .sutra/rules.toml — Architectural constraints for <project>
#
# Severity levels:
#   blocking      — guard denies edits that introduce these
#   advisory      — warning in review, agent proceeds
#   informational — tracked in review output, silent otherwise

# ═══════════════════════════════════════════════════════════════════
# <Category name>
# ═══════════════════════════════════════════════════════════════════
# <1-2 line rationale for this group>

[[constraint]]
kind = "forbidden_dep"
from = "..."
to = "..."
name = "..."
severity = "blocking"
```

Formatting rules:
- Group constraints by architectural concern with section headers
- Include `name` on every constraint (human-readable, kebab-case)
- Include `provenance` when an ADR or design doc motivates the rule
- Include `severity` explicitly — don't rely on defaults
- Add `scope` for `no_cycles` and scoped rules
- Comments explain the *why*, not the *what*

Convention section at the end:
```toml
[conventions]
suppress = [
  "<id>",  # <what it says — so future readers know>
]
```

### 6. Set up .gitignore

Check the project's `.gitignore` for `.sutra/` handling:

- If `.sutra/` is fully ignored, update to ignore workspace subdirs but track config:
  ```
  .sutra/*/
  .sutra/acks/
  !.sutra/rules.toml
  !.sutra/owners.toml
  ```
- If `.sutra/` is not mentioned, add the above pattern
- If it's already correctly configured, skip

### 7. Verify

Trigger a reparse so sutra picks up the new rules:

```
sutra_parse(workspace="<name>")
```

Then check for violations:

```
sutra_constraints(workspace="<name>", action="list")
sutra_constraints(workspace="<name>", action="violations")
```

Present the results to the user:
- Total constraints loaded (confirms rules.toml parsed correctly)
- Any violations found, organized by severity
- Whether any blocking violations exist (these would block agent edits via the guard)

If blocking violations exist in the current code, discuss with the user:
- Waive them (known debt, tracked with rationale)
- Downgrade to advisory (document but don't block)
- Fix them (if the violation is a real problem)

### 8. Commit

Commit the rules.toml and .gitignore changes as a single commit:

```
Add architectural constraints for <project>

N constraints covering <summary of categories>.
Suppress M tautological FCA conventions.
```

### 9. guard setup

If the project uses Claude Code and the constraints include blocking rules, offer to set up the sutra-guard hook. Show the user what to add to `.claude/settings.json`:

```json
{
  "hooks": {
    "PreToolUse": [{
      "matcher": "Edit|Write",
      "hooks": [{
        "type": "command",
        "command": "sutra-guard"
      }]
    }]
  }
}
```

Only suggest this — don't write it without explicit approval, since it changes the agent's edit behavior.

## Reference: constraint kinds

| Kind | Required fields | Default severity | Purpose |
|---|---|---|---|
| `forbidden_dep` | `from`, `to` (glob patterns) | blocking | Prevent specific import paths |
| `boundary` | `from_component`, `to_component` | blocking | Prevent cross-component imports |
| `no_cycles` | `scope` (path prefix) | blocking | Ban import cycles within a subtree |
| `max_fan_in` | `target` (file path), `threshold` | advisory | Alert when a file exceeds N importers |
| `forbidden_external` | `crates` (name globs); optional `from` (path glob, default `**`), `include_dev` | blocking | Forbid external crates/packages within a scope. Checked from import paths AND Cargo.toml `[dependencies]` |
| `confined_external` | `crates`, `allowed_in` (path globs; `[]` = banned everywhere); optional `include_dev` | blocking | External crates importable ONLY from listed paths (single-point-of-contact rules) |
| `forbidden_pattern` | `language`, `query` (tree-sitter S-expression); optional `scope`, `include_tests` | advisory | AST-pattern enforcement — coding discipline rules. Guard uses introduced-only semantics (pre-existing matches grandfathered). See `vidhi/language-rules/` catalog for vetted rules per language |

Note: in a multi-crate Cargo workspace, sibling-crate imports resolve to real
edges (since sutra/needs-designing/15), so crate-to-crate seams use
`forbidden_dep`/`boundary`. Naming a workspace member in an external kind's
`crates` is a hard error.

### Test code is excluded by default

Every constraint kind skips test-only code unless it opts in. For Rust that
means anything under a `#[cfg(test)]` item or a `#[test]`/`#[tokio::test]`
function — the attribute line through the end of the item it annotates:

- `forbidden_pattern`: matches inside those ranges are dropped.
- `no_cycles` and `forbidden_dep`/`boundary`: `use` statements inside those
  ranges do not create graph edges, so a cycle or violation that exists only in
  test wiring is not reported.

This is the same call clippy makes with `allow-unwrap-in-tests`. Without it a
rule like `no-unwrap` is unusable in idiomatic Rust: on yojana it produced 413
matches, every one of them in an inline test module, burying the production
signal completely.

Opt back in per constraint when the rule genuinely should govern test code —
say, banning a deprecated test helper:

```toml
[[constraint]]
kind = "forbidden_pattern"
language = "rust"
query = '(call_expression function: (identifier) @f (#eq? @f "legacy_fixture")) @match'
name = "no-legacy-fixture"
include_tests = true
```

`include_tests` is not part of constraint identity, so toggling it keeps
existing waivers and ratchet registrations attached.

Two limits worth knowing:

- **`cfg` predicates are read conservatively.** `#[cfg(test)]` and
  `#[cfg(all(test, ...))]` count as test scope; `#[cfg(not(test))]` and
  anything naming a string (`#[cfg(feature = "test-helpers")]`) stay
  production. When in doubt sutra treats code as production — it would rather
  report a match you dismiss than hide one.
- **Rust integration tests (`tests/*.rs`) are not detected**, since they carry
  no `cfg(test)` attribute. Scope rules to `src/` (as the `vidhi` catalog does)
  or waive per file.

Edge-based kinds read the flag from the index, so a workspace indexed before
this landed keeps the old behaviour until it is reparsed. Pattern kinds parse
from disk and take effect immediately.

## Reference: convention lifecycle

| State | Meaning | Agent behavior |
|---|---|---|
| `descriptive` | Pattern exists (auto-detected) | Neutral — no enforcement |
| `preferred` | Pattern should continue | Agent follows it; violations flagged in review |
| `deprecated` | Pattern should fade | Agent avoids it; matches flagged in review |
| `forbidden` | Do not copy this pattern | Agent actively avoids; matches are violations |

Promote via `sutra_conventions(action="set_lifecycle", convention_id="...", lifecycle_state="preferred", reason="...")`.
