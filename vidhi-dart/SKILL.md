---
name: vidhi-dart
description: Dart/Flutter standards baseline — house opinions on null safety, type design, state shape, const discipline, async hygiene, and layering, with exemplars from house projects. Load as the review baseline for Dart diffs, include in subagent briefings for Dart/Flutter implementation, and consult from vidhi-types-first.
---

# vidhi-dart

House Dart/Flutter standards. Each opinion notes where it is enforced: **[analyzer]** = the shared `analysis_options.yaml` baseline (canonical copy in this directory — strict modes + curated lints), **[rule: name]** = sutra forbidden_pattern from `vidhi/language-rules/dart.toml`, **[prose]** = judgment — review catches it. Canonical code lives in [exemplars.md](exemplars.md) — prefer pointing at an exemplar over explaining from scratch.

## Null safety

- No `!` bang assertions: use `??` with a default, `?.`, an early-return guard (promotion does the rest), or Dart 3 patterns (`if (value case final v?)`). A bang trades a compile-time check for a runtime crash. [rule: no-bang-null-assertion]
- Enum-keyed total tables: prefer an exhaustive `switch` expression — the compiler then enforces totality when a variant is added. Where a `Map` is genuinely the right shape, look up via a helper that throws a named `StateError('missing $key in <table>')` — the Dart analog of Rust's `.expect("invariant: ...")` — never a bare `map[key]!`. (Arrow's ~55 `map[key]!` lookups predate this rule; don't imitate them.)
- `late` only when initialization is structurally guaranteed before first access (`initState`, DI setup); it defers null errors to runtime everywhere else. Nullable-with-guard beats `late` when in doubt. [prose]
- Guarded bangs inside collection-if (`if (x != null) ...[x!]`) — hoist to a local instead so flow analysis promotes it (see `overlay_shell.dart` blemish note in the exemplars).

## Types

- No `dynamic` escape hatches; strict-casts/strict-inference/strict-raw-types make the analyzer refuse implicit ones. [analyzer strict modes + `avoid_dynamic_calls`; rule: no-dynamic-type]
- `Map<String, dynamic>` is allowed exactly one place: the `fromJson` boundary. Cast to concrete types there; nothing downstream sees `dynamic` (see `chart_service.dart` in the exemplars).
- Records `(String, int)` over single-use DTO classes for multiple returns; destructure with patterns. [prose]

## State shape

- Kinds-with-payloads are sealed hierarchies switched exhaustively — no `default` branch, so adding a variant breaks every use site at compile time (`time_uncertainty.dart` is the 47-line canonical form).
- No boolean flag soup: `isLoading`/`hasError`/`hasData` booleans make impossible states representable. Mutually exclusive phases are one sealed type (see the `birth_form.dart` counter-example — mounted-guard hygiene does not fix a wrong state shape).
- Async operations that can be cancelled, drained, or fail return a sealed outcome the caller must switch on (`ScanOutcome<T>` in the exemplars) — never `Future<T?>` where null means three things.
- State classes: all-final fields, const constructor, `copyWith`. Hand-written state should also carry `==`/`hashCode` (or use freezed); relying on identity compare is a known house gap — fix it in fresh code. [prose]

## Const and immutability

- `const` constructors on every widget and value type where fields allow; `const` on every static subtree — it is the rebuild-propagation firewall, not a style nicety. [analyzer `prefer_const_*`]
- `final` locals and for-each bindings by default. [analyzer `prefer_final_locals`, `prefer_final_in_for_each`]
- Const data tables as records/lists in calc code (`nabhasa_yoga.dart`), not built imperatively at runtime.

## Collections and cascades

Collection-if/for, spreads, and cascades (`..`) replace imperative build-then-mutate boilerplate — declarative construction, and const-compatible where the elements are const. [analyzer `cascade_invocations`, `prefer_spread_collections`, `prefer_if_elements_to_conditional_expressions`]

## Async

- Every `Future` is awaited or explicitly `unawaited(...)` — fire-and-forget is a decision, stated in code, not an accident. [analyzer `unawaited_futures`]
- After any `await` in widget code: `if (!mounted) return;` before touching `context` or `setState`. [prose]
- Subscriptions, controllers, and timers are cancelled/closed in `dispose`; prefer framework-managed lifecycles (builders, Riverpod `ref.onDispose`) over manual `.listen()`. [prose]
- Catch specifically (`on AuthException catch`), never bare `catch (e)` — and never catch `Error` subtypes; those are bugs. [analyzer `avoid_catches_without_on_clauses`]

## Layering (Flutter)

- Widgets are presentation: layout, animation, routing. Business logic, API calls, and persistence live in injected services/notifiers — a widget constructing its own HTTP client or geocoder is the counter-example shape. [prose]
- Services own external I/O and return typed domain results with typed failures (`ChartApiException`), not raw status codes or exception strings. One retry/refresh helper, not per-call copies.
- No specific state-management framework is mandated (house code uses Riverpod, provider, and plain setState); the invariants above — immutable state, sealed phases, injection, scoped rebuilds — apply to all of them.

## Lints and suppression

- The shared `analysis_options.yaml` baseline is the floor: `package:lints`/`flutter_lints` recommended set, the three strict modes, and the curated rules. Run `dart analyze` / `flutter analyze` before claiming done.
- Never add `// ignore:` or `// ignore_for_file:` to quiet a diagnostic — fix it, or waive via sutra with rationale. Dart has no `#[expect]` analog, so there is no self-expiring middle path. Generated files (`*.g.dart`, `*.freezed.dart`, protobuf output) are excluded from analysis and batch-waived in sutra. [rule: no-ignore-comments]

## Testing

Methodology belongs to `vidhi-tdd`; the Dart mechanics:

- Unit tests in `test/` mirroring `lib/`; widget tests via `testWidgets` + `pumpWidget`; no `pumpAndSettle`-free timing flakiness.
- Fakes over mocking frameworks — most house seams are pure functions over data (`nabhasa_yoga.dart` tests need only literal chart data) or injected interfaces a hand-rolled fake covers.
- Bangs and casts are fine in tests; the no-ignore rule still applies.

## How this skill is used

- **Review**: vidhi-review loads it as the baseline for Dart diffs.
- **Delegation**: subagent briefings for Dart/Flutter implementation say "read vidhi-dart/SKILL.md and exemplars.md first" — this file is the model-portable statement of house style.
- **Design**: vidhi-types-first's checklist is the design-time projection of these standards.

---

Distilled 2026-07 from third-party material in `~/soft/dart-flutter-skills` (ECC dart-flutter-patterns + code-review checklist, flutter-apply-architecture-best-practices) plus house exemplars. Conflicts were resolved in coding_discipline's favor — notably the sources' own examples use bang assertions (`_cachedUser!`, `state.pathParameters['id']!`) that the discipline forbids; the MVVM/repository layering survives, the bangs do not.
