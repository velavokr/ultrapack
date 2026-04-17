---
description: Orchestrate the full ultrapack workflow — slug, task file, design, plan, execute, verify, review. Size-aware, resume-ready. Prefix args with `handsoff` for hands-off mode (fewer prompts, conservative defaults, decision log).
---

# /up:make

Drives a task through the full ultrapack workflow: one task file at `docs/tasks/<slug>.md`, evolving through Design → Plan → Conclusion. Each stage is a separate skill. You orchestrate; the skills do the work.

## Arguments

The user's description of the task follows the command. May be a one-liner ("fix the flaky login test") or a paragraph. Use it as the seed for the slug and the initial framing for `up:udesign`.

**Hands-off activation:** if the first whitespace-delimited token of the arguments is the literal string `handsoff`, enable hands-off mode. Strip that token before deriving the slug or framing for design. Any other spelling (`hands-off`, `handsOff`, `--handsoff`) is treated as part of the description — only the bare token `handsoff` activates. See `## Hands-off mode` below for behavior.

## Flow

### 1. Slug

Propose a short kebab-case slug from the description (e.g. "fix-flaky-login-test"). Confirm with the user before creating anything. If they rename, use their version.

### 2. Resume check

Before creating a new task file, check if `docs/tasks/<slug>.md` already exists.

- **Exists:** read `**Status:**` from the header. Resume from the next stage:
  - `design` → continue design
  - `planning` → run `up:uplan`
  - `executing` → run `up:uexecute`
  - `reviewing` → run `up:ureview`
  - `done` → ask the user what they want to do (start a follow-up, re-open, view conclusion)
- **Doesn't exist:** proceed to step 3.
- **Multiple in-flight tasks:** if more than one `docs/tasks/*.md` has Status ≠ `done`, list them and ask which one the user means (or whether this is a new task).

### 3. Create task file

Create `docs/tasks/<slug>.md` from the template. Status = `design`. Branch = `main` (placeholder until step 5). No worktree. Mode = `hands-off` if the keyword was present, else `interactive`.

Template:

```markdown
# <Task Title>

**Status:** design
**Branch:** main
**Worktree:** none
**Mode:** <interactive|hands-off>

## Design
<empty — filled by up:udesign>

### Invariants
<empty>

### Principles
<empty>

## Plan
<empty — filled by up:uplan>

## Verify
<empty — filled by up:uverify>

## Conclusion
<empty — filled by up:ureview>

### Hands-off decisions
<empty — populated only when Mode is hands-off>

### Deferred (needs user input)
<empty — populated only when Mode is hands-off and a choice had no conservative default>
```

### 4. Size classification

Based on the task description, classify size:

- **Trivial** — one-line change, typo, rename. Skip Design and Plan. Go straight to Execute. Status file still created.
- **Small** — single file or single concept change. Skip Design. Plan runs.
- **Medium / Large** — full flow.

**Interactive mode:** confirm the classification with the user before skipping any stage. When unsure, ask.

**Hands-off mode:** do not confirm. Default to **Medium** (full flow) unless the scope is unambiguously Trivial (true one-liner in one file). Never auto-pick Small or auto-skip Design — Design is the one interactive stage preserved in hands-off. Append the choice to `## Conclusion → ### Hands-off decisions` as `- size: <classification> — <rationale>`.

### 5. Design stage (unless skipped)

Invoke `up:udesign`. It populates `## Design`, `### Invariants`, `### Principles`, and records `TDD: yes / no (reason)`. Status → `planning`.

### 6. Branch & worktree decision

After Design (or immediately for trivial/small tasks), decide:

- **Complex / long-running / touches many files** → suggest a dedicated branch + worktree. Use `up:git-worktrees`.
- **Easy fix / small scope** → suggest working on current branch (usually `main`).

**Interactive mode:** always confirm with the user. Often they want to work directly on main — that's fine.

**Hands-off mode:** default to branch = `main`, worktree = none. Do not suggest a worktree. Log the choice to `## Conclusion → ### Hands-off decisions`. If the task is genuinely long-running (clear from the Design), log that to `### Deferred (needs user input)` with "- worktree: task looks long-running but hands-off defaulted to main — user may want to move this to a worktree" and proceed.

If a branch is created, update the task file's `**Branch:**` and `**Worktree:**` headers.

### 7. Plan stage (unless skipped)

Invoke `up:uplan`. It populates `## Plan`. Status → `executing`.

In hands-off, `up:uplan` auto-proceeds to `up:uexecute` without an approval prompt. It logs `- uplan: plan auto-approved (hands-off)` to `### Hands-off decisions`.

### 8. Execute stage

Invoke `up:uexecute`. Implements the plan, commits incrementally.

### 9. Verify loop

Invoke `up:uverify`. On failure: `up:uverify` describes how each failure *should* have worked, control returns to `up:uexecute`. Loop until verify passes.

### 10. Review stage

Status → `reviewing`. Invoke `up:ureview`. It dispatches `up:ureviewer`, processes findings, fills `## Conclusion`. Status → `done`.

Once the task is concluded as `done`, run the docs-refresh check (see below).

**Review is never skipped**, regardless of size.

### 11. Finish

**Hands-off mode — first:** print the `## Conclusion → ### Hands-off decisions` list (and `### Deferred (needs user input)` if non-empty) to the user and ask verbatim: "Here's what I did to make it hands-off. Want to change anything?" Wait for the user's response before continuing. This is the only required interaction after Design.

Then (both modes) present options to the user:
- Merge / open PR (if on a branch)
- Clean up worktree
- Move on

Execute only after the user chooses.

## After task is done — docs refresh

Run this once, after Review concludes the task as `done` (not after every stage). Scan the project docs and update them if the work surfaced something they should reflect. Cheap, light-touch; not a full doc pass.

Files to scan:
- `CLAUDE.md` (project-wide agent guidance)
- `README.md`
- `docs/**/*.md` (project documentation, excluding the task file itself and archived tasks)

What to look for:
- New conventions, invariants, or principles that should be global → update `CLAUDE.md`
- New components, commands, or features the README should mention
- Stale content contradicted by the stage's work → delete or correct
- Pointers to the task file if future contributors would benefit

Rules:
- If nothing needs updating: say so in one line and move on. Do not invent edits.
- If updates are needed: make them directly, then summarize what changed in 1-3 lines (e.g. "README: fixed install instructions; CLAUDE.md: no change"). Do not prompt for approval first. Do not produce a detailed diff — the user will git-diff if they want.
- Follow `up:udocument`: lead with why, lists over tables, no aspirational content, kill stale content.
- Do not duplicate content across task file and project docs — pick one home per fact.

## Stop conditions

Stop and ask the user when:

- Slug conflicts with an existing task file and resume intent is ambiguous
- Size classification is genuinely unclear (interactive only; in hands-off, default to Medium)
- User has expressed a preference (branch, scope, TDD) that conflicts with the auto-inference
- Any stage's skill returns a blocker

## Rules

- Never skip Review (both modes)
- Never auto-merge or auto-push — the user chooses at step 11 (both modes)
- Never create a worktree without confirming (overridden in hands-off: default is no worktree, never auto-create one)
- Keep the task file as the single source of truth — each stage reads it, each stage writes to it
- External spec / design docs (e.g. `docs/superpowers/specs/*.md`) are read-only during execute. If a stage finds the spec is wrong, surface it to the user — don't mutate it silently
- Don't assume prior session memory — the next agent may be a fresh context reading only the task file
- In hands-off, never invent a default for an ambiguous argument. If a choice has no obvious conservative default and the user didn't specify, log under `### Deferred (needs user input)` in Conclusion and skip that work — do not guess

## Hands-off mode

Activated by prefixing `/up:make` arguments with the literal token `handsoff`. Goal: run the full workflow with the fewest possible user prompts after Design, while making the least assumptions.

**Propagation.** The task-file `**Mode:**` header is the single source of truth. Every child skill reads it. No env var, no config flag, no parameter passing through the Skill tool.

**What hands-off suppresses:**
- Confirmation prompts for size classification, branch, worktree, and TDD (step 4 + step 6).
- The plan-approval gate in `up:uplan` (step 7 auto-proceeds to execute).
- The "should I fix this?" prompt in `up:ureview` for high-confidence actionable findings (they're fixed directly).

**What hands-off preserves:**
- The Design stage's dialogue (clarifying questions still allowed when genuinely blocking).
- `up:uexecute`'s fail-fast rules and stop-and-ask list (ambiguous plan, missing dep, would-invent-fallback). These *are* the "genuinely impossible without user input" exception.
- The verify loop (fail → execute, pass → review).
- The review stage's existence — Review is never skipped in any mode.

**Decision log.** Every auto-choice that would normally have prompted the user is appended to `## Conclusion → ### Hands-off decisions` as `- <stage>: <choice> — <rationale>`. At step 11 this list is printed to the user with "Here's what I did to make it hands-off. Want to change anything?" — giving them one review point at the end instead of many prompts along the way.

**Deferred log.** When a choice has no conservative default (e.g. a required argument with no sensible fallback), do not guess. Append to `### Deferred (needs user input)` with what was skipped and why. The task completes with those items open for the user's attention.

**No-default rule.** Conservative ≠ inventive. In hands-off, "conservative" means *fewer assumptions*, not *make up a safe-looking placeholder*. If the plan or design is silent on a required value and no obvious conservative default exists (e.g. the rigid path of "same as before" or "as the user wrote it"), skip it and defer.

## Terminal state

Task file Status = `done`, Conclusion filled, user has chosen a finish action (merge, PR, cleanup, or defer). In hands-off, the user has also reviewed the `### Hands-off decisions` list.
