# Dart exemplars

Canonical files from house Dart/Flutter projects. When explaining a standard, point at one of these instead of writing a fresh example — they carry the house style in context, with the trade-offs visible. All verified against the standards at selection time (2026-07). House projects are of mixed maturity: where a file has a blemish, the entry says exactly what to imitate and what not to.

## `/home/josh/nhs/soft/astrology/arjuna/arrow/core/lib/src/time_uncertainty.dart` — sealed hierarchy, done small

The whole discipline in 47 lines: a `sealed class TimeUncertainty` with three const-constructor variants (`ExactTime`, `PeriodTime`, `UnknownTime`), constructor `assert`s encoding the domain ranges, and `sampleTimes` as an exhaustive switch *expression* with pattern destructuring (`PeriodTime(:final startHour, :final endHour)`) and a collection-for inside a switch arm. No default branch — adding a variant breaks the switch at compile time. Imitate: this exact shape whenever state has "kinds with different payloads"; never a `String kind` plus nullable fields.

## `/home/josh/nhs/soft/astrology/arjuna/arrow/calc/lib/src/vedic/nabhasa_yoga.dart` — pure functions over data

A 600-line calculation module with zero I/O and zero mutable shared state: static methods take a `Varga`, return `List<YogaResult>`. Yoga tables are `const` lists of records destructured in collection-fors (`for (final (name, houses) in trines)`), map lookups fall back with `?? const []` instead of `!`, and results build via spreads (`[...ashrayaYogas(varga), ...]`). The `YogaResult` sealed base with `NabhasaYoga`/`MahapurushaYoga`/`SolarLunarYoga` subclasses keeps heterogeneous results typed. Imitate: this shape for any calc/analysis logic — pure inputs, owned outputs, const data tables as records, and the tests need only literal chart data.

## `/home/josh/nhs/soft/astrology/foundation/innerorbits/lib/astro/ephemeris_service.dart` — typed outcomes at an async boundary

`ScanOutcome<T>` is a sealed result type distinguishing `ScanOk`/`ScanDrained`/`ScanAborted`/`ScanFailed(error, stack)` — cancellation and failure are values the caller must switch on, not exceptions that vanish into a raw catch. `Cancellable` + `RefCancellable` tie async work lifetime to Riverpod disposal (`ref.onDispose`), with the late-registration case handled (a handler added after cancellation fires immediately). The abstract `EphemerisService` plus conditional import (`if (dart.library.js_interop)`) keeps platform split behind one interface. Imitate: the sealed-outcome pattern for any operation that can be cancelled, drained, or fail — never `Future<T?>` where null means three different things.

## `/home/josh/adityas/explore/lib/api/chart_service.dart` — service layer and typed JSON boundary

An HTTP repository the widgets never see past: injected `http.Client` and `TokenProvider` (testable without a network), a private `_request` helper that owns the 401-refresh-retry loop once, and `ChartApiException(message, statusCode)` as the typed failure — callers catch a domain error, not a status code buried in a string. JSON is cast to concrete types at the boundary (`json['id'] as String`, `DateTime.parse(...)` in `SavedChartSummary.fromJson`) so `dynamic` never leaks past the parse. Imitate: `Map<String, dynamic>` is allowed exactly one place — the `fromJson` constructor — and everything downstream is fully typed.

## `/home/josh/nhs/soft/astrology/foundation/innerorbits/lib/providers/settings_provider.dart` — immutable state, injected

`SettingsState` is all-final with a const constructor and `copyWith`; derived views (`planetOrder`) are exhaustive switch expressions over an enum, not if-chains. `SettingsNotifier` loads prefs defensively — `?? 0` defaults, `.clamp()` on stored indices, unknown enum names rejected in `_loadCustomOrder` — so corrupt persistence degrades to defaults instead of crashing. Persistence lives in the notifier; widgets only read state and call intent methods. Blemishes to not imitate: no `==`/`hashCode` (arrow uses freezed for that; here identity-compare is relied on), the prefs writes are fire-and-forget unawaited futures, and `catch (_) {}` in `_loadCustomOrder` would be better as `Planet.values.asNameMap()[n]`. Imitate the state shape and the clamped decode; fix those three when writing fresh code.

## `/home/josh/adityas/explore/lib/ui/overlay_shell.dart` — const widget discipline, logic-free

A `StatelessWidget` shell that is pure presentation: all-final fields, const constructor, `const` on every static subtree (`const SizedBox(width: 8)`, `const EdgeInsets.fromLTRB(...)`) so rebuild propagation stops at them, and optional header slots composed with collection-if plus spread (`if (onBack != null) ...[...]`). Behavior enters only as `VoidCallback`s; there is no state, no service call, no conditional business branch in `build`. One blemish: `headerLeading!` inside the collection-if — safe because guarded on the previous line, but the discipline answer is hoisting to a local so flow analysis promotes it. Imitate: this slot-based shell shape for any reusable chrome, and the const-everywhere habit.

## Counter-example: `/home/josh/adityas/explore/lib/ui/birth_form.dart`

An 1100-line `StatefulWidget` carrying `_searching`, `_resolving`, `_chartSaved`, `_advancedExpanded` as independent booleans mutated across ~29 `setState` calls, with geocoding, place resolution, and chart-save API calls living directly in the widget state. The line-level hygiene is actually good — `if (!mounted) return;` after every await — which is precisely the lesson: mounted guards don't fix a state shape where `_searching && _resolving` is representable but meaningless. The discipline answer is a sealed `FormPhase` (idle / searching / resolving / saved) driven by an injected service, with the widget switching exhaustively over it. Kept here as the reminder that flag booleans are the Dart equivalent of stringly-typed dispatch.
