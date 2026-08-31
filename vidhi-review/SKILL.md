---
name: vidhi-review
description: Review code changes for one or more yojana tasks. Pulls task context, assembles the diff, and either reviews inline or produces a self-contained brief for external models (codex, opencode, etc.). Use for bounded reviews of specific work — not whole-project audits (use vidhi-release-review for those).
---

# Task Review

Review code changes scoped to one or more yojana tasks. Lighter than vidhi-release-review — no pack, no parallel subagents, no full code-intel sweep. The reviewer focuses on whether *this change* is correct, clean, and consistent with the task's acceptance criteria and the project's contracts.

## When to use

- After completing a task or set of related tasks, before moving on
- When you want a second opinion from another model (codex, opencode, glm)
- When reviewing someone else's work against a yojana task spec

## When NOT to use

- **Whole-project audit** — use vidhi-release-review. It builds a full pack with code-intel and runs parallel architecture + review passes.
- **Architecture improvement** — use vidhi-deepen. This skill checks what's there, not what could be better.
- **Bug investigation** — use vidhi-diagnose. Don't use review to find the bug; review checks whether the fix is right.

## The adversarial pass is load-bearing

First global reflect pass (lessons ledger L5, 2026-06-11): every reviewed
wave across four projects surfaced high-severity findings the implementation
missed, and twice the landed *fix* was itself wrong (a timeout that couldn't
preempt blocking work; a queue write ordered after the cascade that erased
its inputs). Treat the adversarial review as a second design step, required
before a wave or task closes — not an optional quality gate to skip when
confident. Confidence was present in every one of those cases.

## Entry

Accepts one or more yojana task IDs: `vidhi-review chitta/32` or `vidhi-review chitta/32 chitta/33 chitta/34`.

If no task IDs are given, ask the user. Don't guess from git state.

## Process

### 1. Gather task context

For each task ID:

```
yojana_context task="<id>" shape="review"
```

This returns: description, acceptance criteria, decisions, implementation plan, git/doc context_refs, and neighbor summaries.

Collect the set of acceptance criteria across all tasks — these are the spec the reviewer checks against.

### 2. Determine the diff

The diff is the central artifact. Resolve it in priority order:

1. **Git context_refs on the task(s)** — if tasks have `git:commit` refs, use those SHAs to build the range. For multiple commits: `git diff <earliest-parent>..<latest>`.
2. **Blocked-by task refs** — for review tasks, check the blocked-by tasks' git refs. If those also lack refs, search git log for commit messages containing the task IDs (e.g. `git log --oneline | grep -E 'sutra/(84|85|86)'`). Build the range from earliest-parent to latest.
3. **User-specified range** — if the user provides a commit range or branch comparison (e.g. `main..feature`), use that.
4. **Ask** — if none of the above yields a range, ask the user: "What's the diff? A commit range, a branch comparison, or should I use the working tree?"

Capture the diff as text. For large diffs (>500 lines), note the size — the reviewer should still see all of it, but the orchestrator may want to flag that a release-review would be more appropriate.

### 3. Structural review

Call `sutra_review` (requires analysis tier — call `sutra_tools enable: ["analysis"]` first if not already enabled) to get a complete structural analysis in one call:

- Changed files and symbols
- Transitively affected symbols (blast radius)
- Risk score (0.0–1.0) with per-signal breakdown (blast radius, complexity, churn, hotspot overlap, conventions)
- Constraint violations (DD: dependency cycles, forbidden deps)
- Convention violations (FCA: pattern breaks in changed code)
- Recommended reads ranked by review priority

Pass `diff="<base>..<head>"` using the commit range resolved in step 2 — this is essential when commits are already on main, since the default `diff="branch"` diffs against the merge-base and returns nothing for on-main work. Use `diff="staged"` or `diff="unstaged"` if the task's commits aren't on a branch yet.

This tells the reviewer: how risky is this change, what else might break, and where to focus. The recommended reads list is especially useful — it prioritizes convention violation sites over plain affected symbols.

If sutra is unavailable or the workspace isn't indexed, skip this step and note the gap. The review can proceed without it — structural analysis sharpens the review but isn't required.

### 4. Gather project context

Pull the minimum project context a cold reviewer needs:

- `CONTEXT.md` — domain vocabulary (if it exists)
- Relevant ADRs — any ADR referenced in the task or touching the same area as the diff
- Key type signatures — for functions/types the diff modifies, include their doc comments and public interface
- Language standards baseline — for Rust diffs, load `vidhi-rust/SKILL.md`; for Dart/Flutter diffs, load `vidhi-dart/SKILL.md` as the review baseline (and the skill's `exemplars.md` when a finding needs a canonical counter-shape)

Don't pull the entire project README or CLAUDE.md. The reviewer doesn't need project setup instructions — they need domain contracts and design decisions.

### 5. Choose reviewer

The `--reviewer` flag (or user intent) picks the dispatch path:

- **`claude`** (default) — inline review in this session. Proceed to step 6.
- **`codex`** — dispatch to Codex via its companion script. Proceed to step 7.
- **`brief`** — produce a self-contained review directory for any model. Proceed to step 8.
- **`opencode`** — build the brief, then dispatch it to an opencode model headlessly. Proceed to step 9.

If the user mentions codex, opencode, gemini, or another model by name, that's the reviewer. If they say "get a second opinion" without specifying, ask.

### 6. Inline review (claude)

Review the diff against:

- Acceptance criteria from the task(s)
- Project invariants from CONTEXT.md and ADRs
- Impact analysis (callers that might be affected but aren't in the diff)
- General correctness: error handling at boundaries, state transitions, naming consistency

Produce findings using the standard YAML schema (same as vidhi-release-review — see below). Write them directly in the conversation.

For each finding, check: would this matter to the author, or is it noise? Five sharp findings beat fifteen nitpicks.

After findings, write a **verdict**:

- **approve** — changes are correct and clean. Ship it.
- **approve with notes** — changes are correct, minor items to address at convenience.
- **request changes** — issues that should be fixed before this work is considered done.

Update the task(s) in yojana:

```
yojana_task action=comment id="<id>" text="Review: <verdict>. <1-line summary>" author="agent"
```

If the review surfaces new work, offer to file it as a follow-up task rather than expanding the scope of the reviewed task.

### 7. Codex dispatch

Call the codex companion script directly — don't load the codex plugin's slash command into context.

Build focus text from the task context: acceptance criteria, key decisions, and any specific review concerns from the impact analysis. This enriches Codex's review with task-awareness it wouldn't otherwise have.

**Write the focus text to a file first, then pass it via `"$(cat file)"`. Never inline it directly into the bash command string.** Focus text routinely contains backticks around code identifiers (`` `for niter in 0..2` ``, `` `eclipse_where` ``) and apostrophes (`isn't`, `doesn't`). Backticks inside a double-quoted bash string are command substitution — bash will try to execute `goto`, `retval`, etc. as commands, silently mangling the prompt actually sent to Codex (you'll see spurious "command not found" lines in the output, easy to miss since the script still runs). `"$(cat file)"` sidesteps this: command-substitution output is inserted literally and is never re-scanned for nested backtick/`$` expansion, so the file's content is safe regardless of what characters it contains.

```bash
# 1. Write the focus text with the Write tool (not a heredoc) to a scratch path:
#    /tmp/claude-.../scratchpad/codex-focus-<task>.txt
# 2. Then, in ONE bash call with run_in_background + timeout both set:
node ~/.claude/plugins/codex/scripts/codex-companion.mjs adversarial-review \
  --base <base-ref> \
  "$(cat /tmp/claude-.../scratchpad/codex-focus-<task>.txt)" \
  > /tmp/claude-.../scratchpad/codex-out-<task>.log 2>&1
```

**Important:** The `review` subcommand does not accept custom focus text — use `adversarial-review` when passing task context. Always pass `--base` explicitly; omitting it diffs against main, which produces an empty diff if the reviewed commits are already on main.

**Important:** Codex investigations (reading multiple files, diffing against base, sometimes running snippets to check edge cases) routinely run past the Bash tool's 2-minute default timeout, which kills the process and wastes the work done so far. Set `run_in_background: true` and an explicit `timeout` of at least 600000 (10 minutes) **in the same Bash call that launches the script** — not as a follow-up fix after it times out once. Redirect stdout/stderr to a log file (as above) so the background task's output is capturable; don't rely on the tool call's direct return.

The script returns review text. Parse it into the standard finding schema where possible, and record the raw output as a yojana comment on the task:

```
yojana_task action=comment id="<id>" text="Codex review:\n<output>" author="codex"
```

### 8. Brief mode (any external model)

Build a review brief at `/tmp/review-brief-<project>-<date>/`:

```
review-brief/
  00-instructions.md    — reviewer-prompt.md from this skill directory
  10-tasks.md           — combined task context (criteria, decisions, neighbors)
  20-diff.patch         — the diff as a unified patch
  30-review.md          — sutra_review output, formatted (risk score, impact, violations, recommended reads)
  40-context/           — CONTEXT.md, relevant ADRs, key type signatures
```

The brief is self-contained. An external model reads `00-instructions.md` first, then reviews using the rest.

Tell the user the brief location. They feed it to whatever model they want — opencode, gemini-cli, or a future tool. The skill doesn't need to know the invocation syntax for every model; that's the user's concern. The brief is the contract.

When the external review comes back, reconcile findings with any inline review and update yojana.

### 9. opencode dispatch

opencode runs headlessly via `opencode run` and consumes the brief directory natively — its `read` tool works autonomously (no `--auto`, so the review stays read-only and safe). The brief *is* the contract; opencode is just another consumer of it.

**The brief must live inside the repo, at a gitignored path.** opencode sandboxes its `read` tool to the run directory (`--dir <repo>`) — it cannot read paths outside the repo, so a `/tmp` brief is invisible to it (verified live: a `/tmp` brief failed; a brief inside the repo worked). Any gitignored in-repo directory works — pick one that already exists in the repo's `.gitignore`, or create one. In a Rust repo `target/review-brief-<project>-<date>/` is convenient since `target/` is already gitignored, but there's nothing special about `target/`; the only requirements are in-repo (so opencode can read it) and gitignored (so it doesn't dirty the tree). This overrides step 8's `/tmp` location for the opencode path only.

Note the distinction: the **brief** is read by opencode, so it must be in-repo; the **prompt file** is `cat`'d by *your* shell (not opencode) before launch, so it can stay in `/tmp/claude-.../scratchpad/`.

**First, build the brief** exactly as in step 8, but at the in-repo path above. Then invoke opencode against it.

**Model.** Default `opencode/glm-5.2` (cross-vendor second opinion, distinct from Claude and Codex). The `opencode/` prefix routes to opencode zen (uses zen credits). Override with `--model` — good alternatives: `opencode/gpt-5.3-codex` (codex-tuned), `opencode/claude-opus-5`, `opencode/gemini-3.1-pro`. List with `opencode models`.

**Write the prompt to a file first, then pass it via `"$(cat file)"`** — same hazard as the codex path. Focus/instruction text routinely contains backticks around identifiers and apostrophes; inlined into a double-quoted bash string, backticks become command substitution and silently mangle the prompt. `"$(cat file)"` inserts the content literally without re-scanning it.

The prompt points opencode at the brief and tells it to read `00-instructions.md` first:

```bash
# 1. Write the prompt with the Write tool (not a heredoc) to a scratch path
#    (in /tmp — this file is cat'd by your shell, not read by opencode):
#    /tmp/claude-.../scratchpad/opencode-prompt-<task>.txt
#    Contents, roughly (use the in-repo brief path, relative to --dir):
#      Review the change described in the brief at target/review-brief-<project>-<date>/.
#      Read 00-instructions.md first, then use 10-tasks.md, 20-diff.patch,
#      30-review.md, and 40-context/. Output findings in the YAML schema
#      the instructions specify. Do not modify any files.
#
# 2. Then, in ONE bash call with run_in_background + timeout both set:
opencode run --dir <repo> -m opencode/glm-5.2 --variant high \
  "$(cat /tmp/claude-.../scratchpad/opencode-prompt-<task>.txt)" \
  > /tmp/claude-.../scratchpad/opencode-out-<task>.log 2>&1
```

**Important:** Set `run_in_background: true` and an explicit `timeout` of at least 600000 (10 minutes) **in the same Bash call that launches opencode** — a real review reads several files and runs past the Bash tool's 2-minute default, which would kill the work. Redirect stdout/stderr to the log file (as above); don't rely on the tool call's direct return.

Default output format prints just the final review text — capture the log. (Pass `--format json` instead if you want machine-parseable line-delimited events: `type=="text"` parts carry the review, the closing `step_finish` event carries `tokens` and `cost`.)

Parse the output into the standard finding schema where possible, and record the raw output as a yojana comment on the task:

```
yojana_task action=comment id="<id>" text="opencode (glm-5.2) review:\n<output>" author="opencode"
```

## Finding schema

Same as vidhi-release-review for interchangeability:

```yaml
- id: <stable-slug>
  severity: critical | high | medium | low
  category: correctness | contract | design | slop | performance
  title: <one line>
  location: <file:line | file | "task-wide">
  evidence: |
    <what's there now — quote or paraphrase>
  why: |
    <why it matters>
  recommendation: |
    <one approach>
  confidence: high | medium | low
```

Severity calibration:

- **Critical** — breaks correctness, loses data, violates security. Must fix.
- **High** — will cause problems or hurt the next person here. Should fix.
- **Medium** — real issue, bounded blast radius. Fix soon.
- **Low** — cleanup, naming. Batch into follow-up.

## Multi-task reviews

When reviewing multiple tasks together:

- Combine acceptance criteria into one checklist, grouped by task
- Use one unified diff covering all tasks' commits
- Findings reference the specific task they relate to (in `location` or prose)
- One verdict covers the whole set — if any task has issues, it's "request changes" for the group

This is the right choice when tasks are tightly coupled (shared code paths, same module, sequential steps in a feature). For unrelated tasks, separate reviews are better — don't force unrelated changes into one review just because they happened at the same time.

## What NOT to do

- Don't build a full review pack. That's release-review territory.
- Don't run architecture analysis (hotspots, dead code, file health). Stay focused on the diff.
- Don't re-run build/test/lint — assume the implementer already did (check the task's verification notes). If you doubt it, ask, don't silently re-run.
- Don't review code outside the diff unless impact analysis shows a caller that's now broken. The diff is the scope.
- Don't expand task scope based on review findings. File follow-ups instead.
- Don't auto-approve. Even if everything looks clean, say so explicitly with a verdict.
