---
name: vidhi-brainstorm
description: Collaborative design exploration. Turns a vague idea into a shared understanding through structured dialogue — one question at a time, 2-3 approaches with trade-offs, incremental design approval. Use before vidhi-domain or vidhi-prd.
---

# Brainstorm

Turn an idea into a design through collaborative dialogue.

<HARD-GATE>
Do NOT write code, scaffold projects, invoke implementation skills, or take any implementation action during brainstorming. The output of this skill is a shared understanding — not code, not a spec file.
</HARD-GATE>

## "This is too simple to need a design"

No it isn't. A config change, a single utility, a one-endpoint API — all go through this process. Simple projects are where unexamined assumptions waste the most work. The design can be short (a few sentences), but you present it and get approval.

## Process

### 1. Explore context

Before asking a single question, understand what exists:
- Read relevant code, docs, recent commits
- Check for `CONTEXT.md` — if it exists, use its vocabulary throughout
- Check for ADRs in `docs/adr/` — respect existing decisions

### 2. Scope check

Assess before diving into details: if the request describes multiple independent subsystems, flag it immediately. Don't spend questions refining details of something that needs decomposition first.

If the project is too large for a single design, help decompose into sub-projects: what are the independent pieces, how do they relate, what order should they be built? Then brainstorm the first sub-project.

### 3. Clarifying questions

Ask questions one at a time. Wait for the answer before the next question.

- Prefer multiple choice when the options are knowable. Open-ended when they aren't.
- One question per message. If a topic needs more exploration, break it into multiple questions.
- Focus on: purpose, constraints, success criteria, what "done" looks like.
- If a *fact* can be found by exploring the codebase, look it up rather than asking. The *decisions*, though, are the user's — put each one to them and wait for their answer.

### 4. Propose approaches

When you understand enough to see the shape of the solution, propose 2-3 approaches:

- Lead with your recommendation and why
- Name the trade-offs honestly — don't present a false choice where one option is obviously right
- If there's genuinely only one reasonable approach, say so and explain why

### 5. Present the design

Once an approach is chosen, present the design in sections. Scale each section to its complexity — a sentence if straightforward, a few paragraphs if nuanced.

Ask after each section whether it looks right before continuing. Cover:
- Architecture and component boundaries
- Data model and flow
- Key interfaces
- Error cases that matter
- What's explicitly out of scope

**Design for depth over breadth.** Look for opportunities to extract modules with simple interfaces and rich internals that can be tested in isolation. If the design has many small shallow components with complex wiring between them, the boundaries are probably wrong.

**In existing codebases:** follow established patterns. Where existing code has problems that affect this work (a file that's grown too large, unclear boundaries), include targeted improvements in the design. Don't propose unrelated refactoring.

### 6. Confirm shared understanding

Summarize the design in a few sentences. Confirm with the user that you both understand what's being built, what the approach is, and what's out of scope.

Do not proceed to handoff — or any next step — until the user confirms you have reached a shared understanding.

## Handoff

When the design is confirmed, offer next steps based on what makes sense:

- **vidhi-domain** — if the design introduced new concepts, ambiguous terms, or domain complexity that should be sharpened before documenting
- **vidhi-prd** — if the vocabulary is clear and you're ready to capture the design as a PRD

Don't force both. If the domain is straightforward, go straight to PRD.

## Principles

- **One question at a time.** Don't overwhelm.
- **YAGNI ruthlessly.** Remove unnecessary features from designs.
- **Explore alternatives.** Always propose approaches before settling, unless there's genuinely one option.
- **Incremental validation.** Present sections, get approval, then continue.
- **Be flexible.** Go back and revise when something doesn't hold up.
