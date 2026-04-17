---
description: Orchestrate the full ultrapack workflow — slug, task file, design, plan, execute, verify, review. Size-aware, resume-ready, always confirms branch/worktree with the user.
---

# /up:make

Drives a task through the full ultrapack workflow: one task file at `docs/tasks/<slug>.md`, evolving through Design → Plan → Conclusion. Each stage is a separate skill. You orchestrate; the skills do the work.

## Arguments

The user's description of the task follows the command. May be a one-liner ("fix the flaky login test") or a paragraph. Use it as the seed for the slug and the initial framing for `up:design`.

## Flow

### 1. Slug

Propose a short kebab-case slug from the description (e.g. "fix-flaky-login-test"). Confirm with the user before creating anything. If they rename, use their version.

### 2. Resume check

Before creating a new task file, check if `docs/tasks/<slug>.md` already exists.

- **Exists:** read `**Status:**` from the header. Resume from the next stage:
  - `design` → continue design
  - `planning` → run `up:plan`
  - `executing` → run `up:execute`
  - `reviewing` → run `up:review`
  - `done` → ask the user what they want to do (start a follow-up, re-open, view conclusion)
- **Doesn't exist:** proceed to step 3.
- **Multiple in-flight tasks:** if more than one `docs/tasks/*.md` has Status ≠ `done`, list them and ask which one the user means (or whether this is a new task).

### 3. Create task file

Create `docs/tasks/<slug>.md` from the template. Status = `design`. Branch = `main` (placeholder until step 5). No worktree.

Template:

```markdown
# <Task Title>

**Status:** design
**Branch:** main
**Worktree:** none

## Design
<empty — filled by up:design>

### Invariants
<empty>

### Principles
<empty>

## Plan
<empty — filled by up:plan>

## Verify
<empty — filled by up:verify>

## Conclusion
<empty — filled by up:review>
```

### 4. Size classification

Based on the task description, classify size:

- **Trivial** — one-line change, typo, rename. Skip Design and Plan. Go straight to Execute. Status file still created.
- **Small** — single file or single concept change. Skip Design. Plan runs.
- **Medium / Large** — full flow.

**Always confirm the classification with the user** before skipping any stage. When unsure, ask.

### 5. Design stage (unless skipped)

Invoke `up:design`. It populates `## Design`, `### Invariants`, `### Principles`, and records `TDD: yes / no (reason)`. Status → `planning`.

### 6. Branch & worktree decision

After Design (or immediately for trivial/small tasks), decide:

- **Complex / long-running / touches many files** → suggest a dedicated branch + worktree. Use `up:git-worktrees`.
- **Easy fix / small scope** → suggest working on current branch (usually `main`).

**Always confirm with the user.** Often they want to work directly on main — that's fine.

If a branch is created, update the task file's `**Branch:**` and `**Worktree:**` headers.

### 7. Plan stage (unless skipped)

Invoke `up:plan`. It populates `## Plan`. Status → `executing`.

### 8. Execute stage

Invoke `up:execute`. Implements the plan, commits incrementally.

### 9. Verify loop

Invoke `up:verify`. On failure: `up:verify` describes how each failure *should* have worked, control returns to `up:execute`. Loop until verify passes.

### 10. Review stage

Status → `reviewing`. Invoke `up:review`. It dispatches `up:reviewer`, processes findings, fills `## Conclusion`. Status → `done`.

Once the task is concluded as `done`, run the docs-refresh check (see below).

**Review is never skipped**, regardless of size.

### 11. Finish

Present options to the user:
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
- Follow `up:document`: lead with why, lists over tables, no aspirational content, kill stale content.
- Do not duplicate content across task file and project docs — pick one home per fact.

## Stop conditions

Stop and ask the user when:

- Slug conflicts with an existing task file and resume intent is ambiguous
- Size classification is genuinely unclear
- User has expressed a preference (branch, scope, TDD) that conflicts with the auto-inference
- Any stage's skill returns a blocker

## Rules

- Never skip Review
- Never auto-merge or auto-push — the user chooses at step 11
- Never create a worktree without confirming
- Keep the task file as the single source of truth — each stage reads it, each stage writes to it
- External spec / design docs (e.g. `docs/superpowers/specs/*.md`) are read-only during execute. If a stage finds the spec is wrong, surface it to the user — don't mutate it silently
- Don't assume prior session memory — the next agent may be a fresh context reading only the task file

## Terminal state

Task file Status = `done`, Conclusion filled, user has chosen a finish action (merge, PR, cleanup, or defer).
