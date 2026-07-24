# Rust exemplars

Canonical files from house crates. When explaining a standard, point at one of these instead of writing a fresh example — they carry the house style in context, with the trade-offs visible. All verified against the standards in SKILL.md at selection time (2026-07).

## `/home/josh/soft/manas/sutra/src/error.rs` — error taxonomy

The `thiserror` enum per library boundary, done right. Structured variants carry what the caller needs (`tool`, `constraint`, `next_action` — errors that tell the consumer what to *do*), `#[from]` conversions for infrastructure errors, `&'static str` where owned strings would be waste, and a single `code()` mapping to the wire protocol. Imitate: the `next_action` field pattern for any agent-facing or user-facing error.

## `/home/josh/soft/manas/sutra/src/rules.rs` — data modeling

A pure types/schema module: `Severity` as a serde-renamed enum, `ConstraintKind` as a sum type with struct variants, `Arc<str>` for shared immutable identity strings (cheap clones without clone-driven development), and content-addressed identity (`compute_id` = blake3 over kind + params) so renames don't break references. Imitate: closed sets as enums with `as_str`/`from_str_lossy` pairs, not string constants.

## `/home/josh/soft/manas/sutra/src/parser/adapter.rs` — trait boundaries

`LanguageAdapter` behind a registry: one of the few sanctioned `dyn` uses (a genuinely heterogeneous collection of language implementations). `ParseContext<'a>` bundles related borrows into a view struct instead of parameter sprawl. `FcaAttributeSource` shows default trait methods for optional capabilities. Imitate: the layering (trait in the low layer, extension traits per ADR-0003 for cross-layer capabilities without cross-layer imports).

## `/home/josh/soft/manas/sutra/src/constraints/patterns.rs` — pure functions over borrowed data

`check_forbidden_patterns(&[Constraint], &[(&str, &str)], &LanguageRegistry) -> Vec<ConstraintFinding>`: borrows everything it reads, owns only what it returns, no I/O — which is why its tests need no mocks, just literal source strings. Imitate: this shape for any analysis/transform logic; push I/O to the caller.

## `/home/josh/nhs/soft/astrology/swisseph-rs/src/flags.rs` — newtypes over a C API

`bitflags!` structs (`CalcFlags` etc.) over what C exposes as bare `u32` constants, each flag documented with its C name and its interactions ("Forces `NOABERR | NOGDEFL`"), serde behind a feature gate. Imitate: when porting or wrapping a C interface, every magic integer becomes a typed, documented value.

## Counter-example: `/home/josh/soft/manas/yojana/src/state.rs`

A stringly-typed state machine: statuses are `&str` against a `VALID_STATUSES` array, transitions a `match` over string literals. Well-tested — and the tests are doing work the type system should do. The discipline answer is a `Status` enum with a `transitions(&self) -> &[Status]` method: adding a variant then breaks every non-exhaustive use site at compile time instead of at runtime. Kept here as the reminder that tests don't substitute for types.
