---
name: execute
description: Use to implement a validated plan. Executes task-by-task with incremental commits; dispatches up:explorer when codebase context is needed; updates the plan inline when reality forces deviations.
---

# Execute

Implement the plan in `docs/tasks/<slug>.md`'s `## Plan` section. Commit after each phase (or smaller unit). Stop at blockers — don't guess. The goal is a working change, not a heroic marathon.

## Before starting

1. **Read the full task file** — Design, Invariants, Principles, Plan. Plan is not optional reading.
2. **Scan for blockers in the plan** — ambiguity, missing dependencies, contradictory steps. Raise them now rather than after you've written half the code.
3. **Confirm branching/worktree state** — if the plan mandated a branch or worktree that doesn't exist, create it (or stop and ask).
4. **Create TodoWrite from plan phases/tasks** — one todo per phase or per task, whichever granularity the plan uses.

## Per-phase loop

For each phase in the plan:

1. Mark the phase as `in_progress`.
2. Do the work. Follow the plan. Deviations allowed when reality demands it — note them inline in the `## Plan` section with `<!-- deviation: ... -->`. Do not silently depart.
3. Run what you built (`/up:try` at minimum). Capture evidence.
4. Commit. Commit message format: `<type>: <concise summary>` (e.g. `feat:`, `fix:`, `refactor:`, `docs:`, `test:`).
5. Mark phase `completed`.

## When to dispatch the explorer agent

- You need to understand where a concept lives in the codebase
- You need call-graph / execution-path info beyond what you can read quickly
- You need a list of essential files touched by a feature

Dispatch `up:explorer` with a tight scope. Don't over-use — cheap inline Grep/Read beats a subagent for one-shot lookups.

## When to stop and ask

- A plan instruction is ambiguous or contradicts itself
- A dependency the plan assumes is missing
- A test fails in a way that suggests the plan is wrong
- The verify phase would obviously fail even if you finished

Don't force through blockers. Don't guess. Ask.

## Deviations from plan

If you have to change approach mid-execution:
- Note the deviation inline in `## Plan` (`<!-- deviation: used library X instead of Y because... -->`)
- If the deviation is structural enough to invalidate later phases, stop and update the plan before continuing

## TDD

If the task's Design recorded `TDD: yes`, follow `up:test-driven-development` per task:
- Write failing test
- Minimal implementation to pass
- Refactor
- Commit

If `TDD: no`, skip tests — verification happens in the verify phase.

## Never

- Start on main/master without explicit user consent (if the plan specified a branch)
- Skip commits between phases
- Claim complete without running what you built
- Push to remote without explicit user consent

## Terminal state

All phases done and committed → invoke `up:verify`. Do not skip to review. Do not finish the branch yourself.
