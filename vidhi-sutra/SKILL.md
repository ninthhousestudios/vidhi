---
name: vidhi-sutra
description: Onboard a codebase into sutra — register the workspace, discover architecture, and propose a .sutra/rules.toml with constraints and convention management. Use when bringing a new project under sutra code intelligence for the first time, or when resetting an existing project's constraint rules from scratch.
---

# Sutra Onboarding

Generate a `.sutra/rules.toml` for a codebase by discovering its architecture, analyzing dependency patterns, and deriving constraints from what's actually there. The output is a reviewed, committed rules file — not a template.

This is an interactive skill. You explore, analyze, present findings, and let the user shape the rules before writing.

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

Also explore the repo for architecture docs:

- Look for `docs/adr/` or similar ADR directories
- Read `CONTEXT.md`, `CLAUDE.md`, `README.md` for stated architectural intent
- Check for existing `.sutra/rules.toml` — if present, read it as a starting point

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

#### 3b. Cross-layer couplings

From the dependency graph, identify imports that cross layer boundaries in the wrong direction. Classify each:

- **Architectural debt** — exists for pragmatic reasons, should be advisory constraints to document and create pressure
- **Clean violations** — clearly wrong, should be blocking constraints
- **Legitimate** — the layers don't apply here (e.g., pipeline/orchestrator importing everything is expected)

#### 3c. Isolation boundaries

Look for modules that must stay lightweight or pure. Common patterns:

- **Guard/hook binaries** — real-time tools that can't afford heavy deps
- **Worker threads** — pure computation communicating via channels
- **Shared libraries/crates** — reusable code that shouldn't depend on application-specific modules
- **Plugin/adapter interfaces** — should depend on traits, not implementations

#### 3d. Hub modules

From `sutra_map`, identify files with high fan-in that aren't infrastructure. These are candidates for `max_fan_in` constraints — not to prevent legitimate use but to detect when a module is accumulating too many responsibilities.

Reasonable thresholds:
- Infrastructure (error, config, utilities): no limit needed
- Domain modules: 10–20 depending on codebase size
- Avoid setting limits so tight they trigger on current state — set them ~30% above current fan-in as guardrails

#### 3e. Cycle candidates

Identify directory subtrees where cycles would be architecturally harmful. Good candidates for `no_cycles`:

- Tool/handler directories (each handler should be independent)
- Plugin/adapter directories
- Core domain module directories

Don't add `no_cycles` to the entire `src/` — it's too broad and legitimate reference cycles exist in Rust (e.g., `mod.rs` re-exporting from submodules).

#### 3f. Convention triage

Review the auto-detected conventions. Classify them:

- **Tautological** — "symbols in directory X are in directory X" (component-scoped conventions where the consequent is just `in:<directory>`). Add to suppress list.
- **Structural truths** — "test functions are functions" (`in:tests → kind:function`). Suppress.
- **Meaningful patterns** — "public functions return Result", "methods take &self". Leave active; candidates for promotion to preferred.
- **Potentially wrong** — patterns that exist due to coincidence, not intent. Leave descriptive, monitor.

Also check for pending promotion proposals and flag ones worth accepting.

### 4. Present the constraint plan

Before writing any file, present the full plan to the user organized by category:

**Blocking constraints** — the rules the guard will enforce in real time.
Show each with: kind, from/to (or scope), name, and one-line rationale.

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

### 9. Optional: guard setup

If the project uses Claude Code and the constraints include blocking rules, offer to set up the sutra-guard hook. Show the user what to add to `.claude/settings.json`:

```json
{
  "hooks": {
    "PreToolUse": [{
      "matcher": "Edit|Write",
      "command": "sutra-guard"
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

## Reference: convention lifecycle

| State | Meaning | Agent behavior |
|---|---|---|
| `descriptive` | Pattern exists (auto-detected) | Neutral — no enforcement |
| `preferred` | Pattern should continue | Agent follows it; violations flagged in review |
| `deprecated` | Pattern should fade | Agent avoids it; matches flagged in review |
| `forbidden` | Do not copy this pattern | Agent actively avoids; matches are violations |

Promote via `sutra_conventions(action="set_lifecycle", convention_id="...", lifecycle_state="preferred", reason="...")`.
