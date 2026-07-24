---
name: vidhi-types-first
description: Types-first design gate for Rust and Dart — produce the type skeleton (data types, signatures, error taxonomy, no bodies) and stop for human review before implementing. Use when Josh invokes it on a new module, feature, or subsystem, or asks to design the types together.
---

# vidhi-types-first

In Rust and Dart the types *are* the design: ownership, error taxonomy, dispatch, and which states are representable are all decided in signatures. A 40-line skeleton is cheap to review and grill; a 400-line implementation is not. This skill front-loads the review to where the leverage is — once the skeleton is approved, the compiler drags the implementation into shape.

## When

- A non-trivial new unit: module, subsystem, or feature with a real data model
- A refactor that moves ownership or changes a public surface
- Not for plumbing, one-liners, or code whose shape is already dictated by an existing trait/interface

This is an on-demand gate — Josh decides when to run it. When a task looks like it qualifies and the skill wasn't invoked, suggest it in one line; don't start it unbidden.

## Process

**1. Context sweep.** `sutra_lessons(query=<what you're building>)` for prior constraints; `sutra_explore` the neighboring code the unit must compose with; note local conventions (error crate in use, module layout, naming).

**2. Produce the skeleton.** One artifact. It contains only:

- **Data types** — structs/enums (Rust), sealed classes/records (Dart). Make illegal states unrepresentable: an enum where a bool+Option combination was tempting, a newtype where a raw `String`/`i64` carries invariants.
- **Public signatures** — functions/methods/traits with deliberate borrows (`&str`/`&[T]` params unless ownership is stored), lifetimes, and generics. Minimal `pub` surface.
- **Error taxonomy** — which failures return `Result` (and the error enum's variants), which are invariant panics (`.expect("invariant: ...")`), which are `unreachable!`. Dart: which throw vs return sealed result types.
- **Dispatch decisions** — generics/`impl Trait` by default; `dyn`, `Box`, `Arc<Mutex>` only with the reason stated inline.
- Bodies are `todo!()` / `throw UnimplementedError()` — nothing else.

**3. Type-check the skeleton.** Write it to disk and run `cargo check` / `dart analyze` before presenting — a skeleton that doesn't type-check wastes the review. (The `no-todo-unimplemented` rule is advisory; the todos die in step 5.)

**4. Flag the contested points, then STOP.** End the artifact with 2–4 bullets naming the decisions most worth grilling: ownership choices, panic-vs-Result boundaries, any dynamic dispatch, any unsafe. Then wait — no implementation until the skeleton is approved. Iterate here; this is the cheap place to be wrong.

**5. Implement to the contract.** Approved signatures are the contract. If implementation reveals the skeleton was wrong — a lifetime that doesn't compose, a missing error variant — surface the deviation and get a nod; don't silently absorb it.

## Skeleton checklist

The design-time projection of `vidhi-rust` — consult that skill (and its exemplars) when a checklist line needs the full standard or a canonical example.

Rust:
- No owned params where a slice serves (`String` → `&str`, `Vec<T>` → `&[T]`)
- No `clone()` implied by the design — sharing is `&`/lifetimes, or a stated `Arc` reason
- Error enum per library boundary (`thiserror` idiom); `anyhow`/`.expect` at bin edges only
- Every panic path is a named invariant, not a shrug
- `dyn` / `Arc<Mutex>` absent, or justified where declared

Dart:
- No `dynamic`, no `!` — sealed hierarchies with exhaustive `switch`; `?.`/`??` for nullability
- `const` constructors wherever construction allows
- Sealed result types over exception control flow for expected failures
