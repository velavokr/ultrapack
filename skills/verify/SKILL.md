---
name: verify
description: Use after execute to confirm the change actually works. Builds a checklist of what should and shouldn't hold, runs /up:try-style checks, does a manual smoke test, loops back to execute on failure.
---

# Verify

Evidence before claims, always. If you haven't run the check in this message, you cannot claim it passes.

## Output

Verify does **not** write to the task file directly. Its output is:
- A pass verdict (moves the workflow to `up:review`)
- Or a fail verdict with a list of what failed and how each *should have* worked (loops back to `up:execute`)

## Process

### 1. Build the checklist

From the Plan + Invariants + Design, enumerate:

- **Should-hold checks** (positive): specific behaviors that must now work. "POST /items returns 201 for a valid payload." "`Dataset.load()` reads the file and returns the expected schema."
- **Shouldn't-hold checks** (negative): things that must still fail or be rejected. "`Dataset.load()` raises on a missing file." "Invariant: `Dataset` does not import `training/`."
- **Invariant checks**: every invariant from Design has a corresponding check.

The checklist lives in-session, not in the task file.

### 2. Run each check

For each item:
- Use `/up:try`-style minimal verification. Direct command > scripted harness.
- One-off scripts go in `tmp/` (project-local, gitignored). Naming is agent's choice; clean up after.
- Capture the actual output. "Looks right" is not evidence.
- Decide pass/fail based on what you saw, not what you expected.

### 3. Manual smoke test

Run the shortest end-to-end path that exercises the change:
- CLI change → invoke the command with representative input
- API change → `curl` against a running server
- UI change → open in browser, click through the feature
- ML change → run a tiny training step or inference call

If you can't run the smoke test (e.g. need infra that's not available), say so explicitly. Don't fabricate success.

### 4. Consolidate

- **All passes** → declare verify passed. Terminal state: invoke `up:review`.
- **Any fails** → for each failure, describe how it *should have* worked conceptually. Loop back to `up:execute` with these notes. Do not move forward.

## Future Work vs. incomplete work

When something fails or is ambiguous, you may be tempted to move it to `## Conclusion → Future Work`. Rules:

- **In-scope = do it.** If the plan mandated it, it's not future work. Complete it or explicitly rescope with user consent.
- **Justification required.** Moving an item to Future Work needs a pointer to: (a) a Design-scope line that excludes it, or (b) a new fact discovered mid-execution that changes scope. No hand-waving.
- **The workflow is not a slacking loophole.** "I'll do it later" without justification = keep working.

## Never

- Claim pass without running the check in this message
- Use "should work", "probably works", "looks correct" as verdicts
- Trust an agent's self-report without verifying the diff
- Declare pass when any check failed
- Skip verify to get to review faster

## Red flags — STOP

- "Just this once"
- "I'm confident"
- "Linter passed" (linter ≠ runtime)
- "Tests pass" (tests ≠ smoke test)
- "Agent said done" (verify the diff yourself)

## Terminal state

Pass → `up:review`. Fail → `up:execute` with remediation notes.
