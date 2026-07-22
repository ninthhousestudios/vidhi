# language-rules

Catalog of `forbidden_pattern` constraint rules for sutra, organized by language. Skills (`vidhi-sutra-seed`, `vidhi-sutra-adopt`, `vidhi-sutra-tend`) read these catalogs and propose relevant rules during governance setup.

## How it works

Each `{language}.toml` contains `[[rule]]` entries with:

| Field | Purpose |
|---|---|
| `name` | Constraint name (becomes `name` in rules.toml) |
| `principle` | Which coding_discipline principle this enforces |
| `query` | Tree-sitter S-expression (becomes `query` in rules.toml) |
| `recommended_severity` | Default suggestion (project overrides) |
| `scope_hint` | Suggested scope (project overrides) |
| `provenance_hint` | Default provenance string |
| `description` | What the rule catches (presented to user during adoption) |
| `false_positives` | Known legitimate matches (presented to user) |
| `guidance` | Waive-vs-restructure decision guidance |

When a skill adopts a rule, it writes a `[[constraint]]` block into the project's `.sutra/rules.toml`:

```toml
[[constraint]]
kind = "forbidden_pattern"
language = "rust"           # from the catalog filename
query = '...'               # from rule.query
name = "no-clone-driven-dev"  # from rule.name
severity = "advisory"       # from rule.recommended_severity or user override
scope = "src/"              # from rule.scope_hint or user override
provenance = "CLAUDE.md coding_discipline"  # from rule.provenance_hint
```

## Adding rules

1. Write the tree-sitter S-expression query
2. Validate it against a real project (`sutra_constraints action=list` reports parse errors)
3. Add a `[[rule]]` entry with all metadata fields
4. Note at the bottom of the file which coding_discipline principles remain prose-only and why

## Current coverage

### Rust (2 rules)
- `no-clone-driven-dev` — `.clone()` call detection
- `no-to-owned-bypass` — `.to_owned()` call detection

### Dart (2 rules)
- `no-dynamic-type` — `dynamic` type annotation detection
- `no-bang-null-assertion` — `!` null-assertion operator detection

### Python (1 rule)
- `no-new-returning-never` — `__new__` annotated `-> Never` in a `.pyi` stub

Python rules are evaluated against `.py` **and** `.pyi`. Stubs are parsed for
constraint matching only and never indexed, so stub-scoped rules do not skew
symbol, fan-in or co-change rollups.

### Not yet enforceable (prose-only in coding_discipline)
- Rust: Prefer Slices Over Containers, Avoid Premature Dynamic Allocation, Graph & Tree Semantics
- Dart: Enforce Const Constructors, Cascade & Collection Operators
- Python: stub/runtime namespace parity, `@final` on returned-only pyclasses
