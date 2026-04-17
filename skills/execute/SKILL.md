---
name: execute
description: Use to implement an approved plan. Works from a checklist, commits per phase, verifies branch/worktree correctness, forbids silent fallbacks, dispatches the planner skill when deviations invalidate the plan. Deviations and risks go to the task file's Conclusion.
---

# Execute

Implement the approved `## Plan` from `docs/tasks/<slug>.md`. Work from a checklist, commit after each phase, stop at blockers rather than guess. The goal is a working change that honors Design and Plan.

## Before starting

<required>
1. Read the full task file — Design, Invariants, Principles, Plan. Plan is not optional reading.
2. Scan the plan for ambiguity, missing dependencies, or contradictory steps. Raise now, not after writing half the code.
3. Verify branch + worktree. Check `git rev-parse --show-toplevel` and `git branch --show-current` match the task file's `**Branch:**` and `**Worktree:**` headers. If mismatched: stop and ask.
4. Build the checklist — one todo per plan phase (or per task if phases are coarse). Use TodoWrite.
</required>

## Branch / worktree correctness

<system-reminder>
Editing the wrong repository is one of the most common bugs. Before any write, confirm:

- `pwd` matches the task file's `**Worktree:**` (or the main repo if `none`)
- `git branch --show-current` matches `**Branch:**`

When you dispatch a subagent (`up:explorer`, `up:researcher`), pass the intended working directory explicitly in the prompt. Subagents do not inherit your `cwd` reliably across harnesses.
</system-reminder>

## Per-phase loop

For each phase in the plan:

<required>
1. Mark the phase `in_progress` in TodoWrite.
2. Do the work. Follow the plan exactly, unless reality forces a deviation (see below).
3. Run what you built — `/up:try` at minimum. Capture actual output.
4. Commit. Format: `<type>: <concise summary>` (`feat:`, `fix:`, `refactor:`, `docs:`, `test:`).
5. Mark the phase `completed`.
</required>

Do not batch phases. One at a time. Commits give you a rollback surface and a legible history.

## TDD

If Design recorded `TDD: yes`, invoke `up:test-driven-development` for each unit under test:

- Write the failing test, watch it fail for the right reason
- Write the minimal implementation to pass
- Refactor with tests green
- Commit

If `TDD: no`, skip the test-first loop; verification happens in `up:verify`.

## When to dispatch `up:explorer`

- You need to trace where a concept lives in the codebase
- You need a call graph or execution path beyond what a quick Grep gives
- You need a ranked list of essential files for a feature

Dispatch with tight scope and pass the working directory explicitly. Don't over-use — inline Grep/Read beats a subagent for one-shot lookups.

## Forbidden: inventing fallbacks, defaults, or best-effort behavior

<system-reminder>
No silent fallbacks. No invented defaults. No "best effort" try/except that swallows the error. If you're tempted to write `.get("attr", 0)` and zero is not a genuine, intended default — don't. Crash > corrupt state.

Never add `try: ... except: pass`. Never catch a broad exception to "keep going". Never substitute a placeholder value so the code "works for now".

If the plan is silent on what to do when X is missing, the answer is: let it raise. Then tell the user this is a potential failure point and add it to the task file's `## Conclusion` as a known risk.
</system-reminder>

<bad-example>
```python
# Plan said nothing about missing config. Agent "helpfully" defaults.
timeout = config.get("timeout", 30)
user_id = payload.get("user_id", "anonymous")
```

This hides real bugs. "anonymous" user_ids leak into downstream systems. 30s timeouts mask misconfigured services.
</bad-example>

<good-example>
```python
# Rigid. If it breaks, it breaks loudly.
timeout = config["timeout"]
user_id = payload["user_id"]
```

Then append to the task file's `## Conclusion` under "Known risks":
> `- config["timeout"] will KeyError if config is partial. Plan didn't specify a default; recommend either adding one with user approval, or validating config at load time.`
</good-example>

Raise these potential failure points with the user immediately, not after execute completes.

## Deviations from plan

A deviation is any structural change from what the plan says. File moved to a different location, method signature different, phase ordering swapped, a phase cut, a phase added.

<required>
When a deviation happens:

1. Do not edit the Plan inline. The plan is the contract that was approved; it stays as-is for the review.
2. Record the deviation in the task file's `## Conclusion` under a `### Deviations from plan` subsection (create if missing). Format: `- <what changed> — <why>`.
3. If the deviation is minor (renamed a helper, swapped two steps) — continue execution.
4. If the deviation is structural enough that later phases in the plan no longer apply — stop executing. Invoke `up:plan` with enough context (what was done, what no longer applies, what new reality is). Let the planner skill update the plan before resuming.
</required>

<good-example>
"Phase 3 planned a new `CacheBackend` class. During phase 2 I discovered an existing `cache/backend.py` with a usable interface. Extending it is simpler.

Appended to `## Conclusion`:
> - Phase 3: used existing `cache.backend.CacheBackend` instead of creating a new one.

Continuing to phase 4 — later phases are unaffected."
</good-example>

<bad-example>
Agent edits `## Plan` inline with `<!-- deviation: ... -->` comments. Reviewer now can't tell what the plan *was*, only what it became. Plan-alignment check is compromised.
</bad-example>

## When to stop and ask

- A plan instruction is ambiguous or self-contradictory
- A dependency the plan assumes is missing
- A test fails in a way that suggests the plan is wrong (not just the implementation)
- Verify would obviously fail even after you finish
- You're about to invent a fallback / default / catch-all

Don't force through. Ask.

## Never

- Start on `main`/`master` without explicit user consent if the plan specified a branch
- Skip the commit between phases
- Claim complete without running what you built
- Push to remote without explicit user consent
- Edit the Plan section to hide a deviation
- Invent a silent fallback to avoid stopping

## Terminal state

All phases done and committed → invoke `up:verify`. Do not skip to review. Do not finish the branch yourself.
