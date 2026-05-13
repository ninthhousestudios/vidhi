---
name: vidhi-release-review
description: Run a thorough, multi-pass code review of a project at a release or checkpoint moment. Use when the user wants to verify a project is ready to ship, audit a long upgrade arc, or get fresh-eyes review on accumulated changes. Different from vidhi-architecture (forward-looking refactor candidates) and per-task PR review (which uses yojana_context shape="review" directly).
---

# Release Review

A two-pass review for a project at a release or checkpoint moment. Designed so the same review pack is also the input for external models (codex, glm, others) — no Claude-specific tooling lives in the pack.

## When to use

- Before tagging a version
- Mid-stream during a long upgrade arc (checkpoint review — substitute the verdict, see Severity below)
- Before handing a project to other reviewers (e.g. external models)

## When NOT to use

- **Per-task PR review** — call `yojana_context shape="review"` for the task plus the diff. Different, lighter-weight workflow. The release review's pack is overkill for a single task.
- **Forward-looking refactor planning** — use vidhi-architecture instead. (Release review *spawns* vidhi-architecture as one of its two passes — see step 5.)
- **Cross-project consistency for the manas constellation** — wait for vidhi-constellation-review (TBD).

## Process

### 1. Verify project state

- `git status --short` — uncommitted work in the diff makes the review noisy. If dirty, ask the user whether to stash or include.
- Confirm scope with the user: tagged release, commit range (e.g. `wm-1..wm-12`), or all-of-HEAD?
- Note in `00-meta.md` whether this is a release gate or a checkpoint.
- Check whether a running daemon's version matches the source version — they may differ during a deploy gap. Record both in meta.

### 2. Build the review pack

Pack location: `/tmp/review-pack-<project>-<date>/` (transient — pack is reproducible from source).

Pack tree:

```
review-pack/
  00-meta.md
  10-context/
    README.md
    CLAUDE.md           # project-level only, NEVER user-level
    CONTEXT.md
    Cargo.toml          # or pubspec.yaml, package.json, etc.
    adrs/<all>
    principles.md       # if present
    plans/<relevant>    # PRD, pivot doc, etc.
  20-code-intel/
    SUMMARY.md          # CURATED — see step 3
    raw/<json dumps>    # for the curious; reviewers should use SUMMARY
  30-verification/
    SUMMARY.md          # build/test/lint/fmt status
    cargo-build.txt     # or equivalent
    cargo-clippy.txt
    cargo-fmt.txt
    cargo-test.txt      # or NOT_RUN with reason
```

Source tree stays in place; the pack references it. **Do not copy the source.**

### 3. Curate the code-intel SUMMARY

Don't dump raw sutra JSON to reviewers. Reviewer attention is the constraint.

**Enable the analysis tier first:** Call `sutra_tools` with `enable: ["analysis"]` before any analysis-tier tools. Without this, `sutra_hotspots`, `sutra_file_health`, `sutra_dead`, and `sutra_cochange` will error.

Sutra calls to make:
- `sutra_health` — workspace freshness
- `sutra_outline` on the crate root (e.g. `src/lib.rs`) — module table of contents
- `sutra_hotspots` — top 10-15
- `sutra_file_health` — top 10-15
- `sutra_dead` — filtered (see caveats)
- `sutra_cochange` on top 3 hotspots
- `sutra_review` — diff-specific structural analysis (risk score, changed/affected symbols, constraint and convention violations, recommended reads). Include this as a "Diff Analysis" subsection in the SUMMARY. This pre-focuses reviewers on the diff's structural impact so they spend tokens on judgment, not navigation.

Caveats to bake into SUMMARY:
- **Filter `sutra_dead` for test helpers and MCP-registered handlers.** `sutra/13` added auto-exclusion for `#[test]` functions and `#[cfg(test)]` modules, but helper functions inside test files (`tests/*.rs`) and inside `#[cfg(test)]` modules still appear. Filter those manually. MCP-framework-registered tool handlers (all `src/tools/*.rs` files showing as "unreachable") are also false positives — filter pack-side.
- **`file_health mode=actionable`** (default) correctly filters foundational files since `sutra/11`. No manual annotation needed.

Always include a "what files matter most" table — top 5 hotspots by score with a one-line interpretation. The reviewer focuses there first.

### 4. Verification

Run language-appropriate checks:
- **Rust:** `cargo build`, `cargo clippy --all-targets --no-deps`, `cargo fmt --check`, `cargo test` if test infra is present
- **Dart/Flutter:** `flutter test`, `flutter analyze`, `dart format --output=none --set-exit-if-changed .`
- **Other:** check for Makefile, package.json scripts, etc.

Auto-detect test infrastructure rather than assuming it's missing:
- Rust + Postgres: `TEST_DATABASE_URL` set + `pg_isready` succeeds + the test DB exists in `psql -lqt`
- Rust + ONNX: model files present at expected paths (e.g. `~/.chitta/models/`)
- Dart/Flutter: any device or emulator listed by `flutter devices`

If infra is genuinely absent, mark `NOT_RUN` with a *specific* reason. "no live Postgres at localhost:5432" beats "infra missing." The reviewer needs to know what's flying blind.

### 5. Launch two subagents in parallel

Each gets the pack location, source root, and a tightly-scoped prompt. Run them as parallel `Agent` calls in a single message.

**Subagent A — vidhi-architecture pass.** Prompt them to follow `vidhi-architecture/SKILL.md` against the project, but stop before the grilling loop (no user to grill). Output: `docs/reviews/<date>-<scope>-architecture.md`. They produce deepening candidates only — no slop, no correctness, no naming.

**Subagent B — code review.** Prompt them with `reviewer-prompt.md` (in this skill directory) plus the pack location. Output: `docs/reviews/<date>-<scope>-review.md` with verdict, design section, YAML findings, synthesis, slop list.

Why two passes? They surface non-overlapping findings. Architecture catches structural drift (concept-spread-across-files, missing module concentrations); review catches correctness, contract, and slop. Tested 2026-05-08 on chitta — minimal overlap, complementary root causes.

### 6. Synthesize

Read both review files in full. Write a synthesis at `docs/reviews/<date>-<scope>.md` (no `-architecture`/`-review` suffix). It should:

- State convergent root causes (where both passes hit the same disease from different angles — these are the highest-leverage findings)
- Propose a fix order grouped into **waves** (one commit per wave)
- Mark blocking vs follow-up
- Reference both source files explicitly

The synthesis is what the user reads first. The two source files are appendices to it.

### 7. Triage to yojana

**Don't auto-file.** Present the proposed yojana tasks to the user with severity, suggested wave, acceptance criteria. Wait for confirmation, then file.

Filing structure:
- One yojana task per wave, with findings as `acceptance_criteria` entries
- Tag everything with `review:<date>-<scope>` for traceability back to the review files
- For checkpoint reviews, also tag with the checkpoint identifier (e.g. `wm-checkpoint`, `pre-wm-13`)
- Architectural deepening candidates → `slice_type=HITL` (need user grilling before implementation), other waves → `AFK`

Edge structure:
- Wave A (correctness/blocking) → no upstream deps, ready
- Wave B/C (design/slop) → `relates_to` Wave A
- Architectural deepening → `depends_on` the waves it builds upon

## Severity calibration

Standard release-gate verdicts:
- **Critical** — breaks correctness, loses data, violates security boundaries, unrecoverable state. Must fix before merge.
- **High** — likely to cause production problems or hurt the next person who touches this. Should fix before merge.
- **Medium** — real issue, bounded blast radius. Fix soon, doesn't block release.
- **Low** — cleanup, naming, style. Batch into follow-up.

For **checkpoint reviews** (mid-upgrade), substitute the verdict line:
- continue upgrade as planned
- continue with adjustments
- pause and address issues before proceeding

## What NOT to do

- Don't include prior reviews in the pack. Fresh eyes is the point.
- Don't include the author's framing of the changes (commit messages with "why", PR descriptions, session summaries).
- Don't ask reviewer subagents to do search-style grunt work (find dead code, locate hotspots). The pack pre-focuses them; that's the leverage.
- Don't hand reviewers raw sutra JSON. Curate.
- Don't run cargo before assembling the pack — verification *is* part of pack-build.
- Don't auto-file yojana tasks without user confirmation.
- Don't relitigate ADRs. Flag a contradiction explicitly only when the friction is real.
- Don't pad subagent prompts with "be thorough" or "consider all angles." Sharpness beats coverage. Tell them what to focus on.

## Outputs

Three files in the project's `docs/reviews/`:

- `<date>-<scope>-architecture.md` — architecture pass (vidhi-architecture skill)
- `<date>-<scope>-review.md` — code review pass
- `<date>-<scope>.md` — synthesis (the entry point readers should open first)

Plus the transient pack at `/tmp/review-pack-<project>-<date>/`.

## Scaling beyond Claude

The pack and prompt are model-agnostic. To use codex / glm / etc:

1. Build the pack as above.
2. Hand them the pack directory + `reviewer-prompt.md` from this skill.
3. They produce a markdown output following the same finding schema.
4. The synthesis step (currently a Claude task) reconciles N reviews instead of 2.

**Subagent limitation:** Step 5 assumes the orchestrator can spawn parallel subagents (Claude Code's `Agent` tool). Codex, OpenCode, and other harnesses don't have this capability. Options: (a) run the two passes as separate tasks/sessions and synthesize afterward, (b) have the single agent do both passes sequentially, (c) keep Claude Code as orchestrator and hand the pack to codex/opencode for one of the review passes (e.g. codex does code review while Claude does architecture).

Until the workflow is proven on more than one project, pack-build remains a Claude-orchestrated process driven by this skill rather than a script. If the same orchestration repeats across projects without significant per-project tweaking, port to a `bin/build-review-pack` binary in vidhi/.

## Known caveats (sutra dependencies)

| Sutra issue | Status | Residual workaround |
|---|---|---|
| `sutra/13` (test exemption) | Partial — `#[test]` fns excluded, but test helpers and `tests/*.rs` files still appear | Manual filter in pack-build for test helpers and MCP-registered handlers |
| `sutra/11` (foundational penalty) | Fixed — `mode=actionable` filters these | None |
| `sutra/12` (cochange git bug) | Fixed | None |
