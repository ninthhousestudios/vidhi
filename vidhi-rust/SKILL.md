---
name: vidhi-rust
description: Rust standards baseline — house opinions on errors, ownership, type design, dispatch, concurrency, and lint discipline, with exemplars from house crates. Load as the review baseline for Rust diffs, include in subagent briefings for Rust implementation, and consult from vidhi-types-first.
---

# vidhi-rust

House Rust standards. Each opinion notes where it is enforced: **[clippy]** = the workspace `[lints.clippy]` baseline, **[rule: name]** = sutra forbidden_pattern from `vidhi/language-rules/rust.toml`, **[prose]** = judgment — review catches it. Deeper treatments live in `references/`; canonical code lives in [exemplars.md](exemplars.md) — prefer pointing at an exemplar over explaining from scratch.

## Errors

- Recoverable failures return `Result` and propagate with `?`. Violated invariants panic via `.expect("invariant: ...")` naming what would have to break — bare `.unwrap()` documents nothing. [clippy `unwrap_used`; rule: no-unwrap]
- Errors are for callers, panics are for programmers: if the caller could reasonably act on the failure, it is a `Result` variant, not a panic.
- Each library boundary owns a `thiserror` enum; `anyhow` and `.expect` belong at bin edges. A good error carries what the caller needs to *act* — see `SutraError`'s `next_action` pattern in the exemplars.
- `todo!()`/`unimplemented!()` never survive a task; `unreachable!("why")` documents genuine impossibility. [clippy `todo`/`unimplemented`; rule: no-todo-unimplemented]

Reference: [references/errors.md](references/errors.md)

## Ownership

- Read-only parameters take slices (`&str`, `&[T]`); owned parameters only where the callee stores. Never force the caller to allocate to pass an argument. [prose]
- `clone()` is a decision, not a fix. Legitimate: `Arc` handle sharing, cheap `Copy`, an API that requires owned. Never: appeasing the borrow checker. [rule: no-clone-driven-dev, no-to-owned-bypass]
- Shared immutable strings are `Arc<str>`, not cloned `String`s (see `Constraint.id` in the exemplars).
- Bundle related borrows into a lifetime-carrying view struct (`ParseContext<'a>`) instead of parameter sprawl or defensive copies.

Reference: [references/pointers.md](references/pointers.md), [references/performance.md](references/performance.md)

## Type design

- Make illegal states unrepresentable: an enum where a bool+Option combination was tempting; a newtype where a raw `String`/`u32` carries invariants (`CalcFlags` over bare bits).
- Closed sets are sum types with exhaustive `match` — the compiler then enforces completeness at every use site when a variant is added.
- Typestate when a runtime state check would be a recurring bug class — compile-time protocol enforcement is cheap in Rust. [references/typestate.md](references/typestate.md)
- Derive `Copy` on small value types; derive only what is used.

## Dispatch

Static where you can, dynamic where you must: generics and `impl Trait` by default. `dyn` requires a heterogeneous collection or true runtime polymorphism, and the reason is stated where it is declared — the `LanguageAdapter` registry in the exemplars is the sanctioned shape. [prose]

Reference: [references/dispatch.md](references/dispatch.md)

## Concurrency

- Prefer ownership transfer (channels) and scoped borrows over shared mutable state. `Arc<Mutex<T>>` is a flag, not a default — it needs the same stated justification as `dyn`.
- `lock().unwrap()` is the standard poisoning idiom and is fine; waive the no-unwrap rule for it with that rationale.
- Understand a `Send`/`Sync` bound before adding it; never add one to silence a compiler error you don't understand.

Reference: [references/pointers.md](references/pointers.md)

## Lints and suppression

- The workspace `[lints.clippy]` baseline (`unwrap_used`, `dbg_macro`, `todo`, `unimplemented`, `indexing_slicing`, all warn; tests exempted via `clippy.toml`) is the floor. Run `cargo clippy` before claiming done — plain `cargo build` does not evaluate these.
- Never add `#[allow(...)]` to quiet a lint: fix it, use `#[expect(lint, reason = "...")]`, or waive via sutra with rationale. [rule: no-allow-attributes]
- Every `unsafe` block states the invariant that makes it sound as its sutra waiver rationale. If the invariant is hard to state, restructure toward a safe abstraction instead. [rule: unsafe-requires-waiver]

## Testing

No mandated tooling. Methodology — seams, red-green, anti-patterns — belongs to `vidhi-tdd`; this section is only the Rust mechanics:

- Unit tests in a `#[cfg(test)] mod tests` in-file; integration tests in `tests/`.
- `unwrap`/`expect` are fine (and good) in tests — the clippy baseline exempts them.
- No mocking frameworks unless a seam genuinely demands one; most seams here are pure functions over borrowed data (see `check_forbidden_patterns` in the exemplars) that need no mocks.

## How this skill is used

- **Review**: vidhi-review loads it as the baseline for Rust diffs.
- **Delegation**: subagent briefings for Rust implementation say "read vidhi-rust/SKILL.md and exemplars.md first" — this file is the model-portable statement of house style.
- **Design**: vidhi-types-first's checklist is the design-time projection of these standards.

---

`references/` chapters are adapted from Apollo GraphQL's [Rust Best Practices Handbook](https://github.com/apollographql/rust-best-practices) (MIT). Dropped chapters: idioms (this file + exemplars supersede), clippy config (live workspace baseline supersedes), testing methodology (vidhi-tdd), comments (global comment discipline).
