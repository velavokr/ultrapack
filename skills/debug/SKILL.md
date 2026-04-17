---
name: debug
description: Use when encountering any bug, test failure, or unexpected behavior, before proposing fixes. Enforces root-cause investigation before any change.
---

# Debug

**Iron law:** no fixes without root-cause investigation first. Symptom patches are failure.

## When to use

Any technical issue: test failure, production bug, unexpected behavior, performance regression, build break, integration failure. Especially when:
- Under time pressure (makes guessing tempting)
- You already tried 1+ fixes
- The problem seems "simple" (simple bugs still have root causes)

## Four phases — complete each before the next

### Phase 1 — Reproduce and trace

1. **Read the error completely.** Stack trace, file, line, codes. Don't skim.
2. **Reproduce.** Exact steps. Every time? Sometimes? If not reproducible → gather more data, don't guess.
3. **Check recent changes.** `git diff`, recent commits, new deps, config drift, env differences.
4. **Instrument component boundaries** (multi-component systems):
   - Log what data enters and exits each boundary
   - Run once to see *where* it breaks before deciding *what* to fix
   - Example: workflow → build script → signing script → actual sign; log env vars at each boundary
5. **Trace data flow backward.** When the error is deep in the stack:
   - Where does the bad value originate?
   - What called this with the bad value?
   - Keep tracing up until you find the source. Fix at source, not symptom.

### Phase 2 — Pattern analysis

1. **Find working examples** in the same codebase. What works that's similar?
2. **Read reference implementations completely.** Don't skim. Don't adapt what you didn't fully understand.
3. **List every difference** between working and broken. Don't assume "that can't matter."
4. **Enumerate dependencies and assumptions** — configs, env, adjacent components.

### Phase 3 — Hypothesis and minimal test

1. **Form a single hypothesis.** Write it: "I think X is the root cause because Y."
2. **Test minimally.** Smallest possible change. One variable at a time.
3. **Verify or discard.** Confirmed → Phase 4. Not confirmed → new hypothesis. Don't stack guesses.
4. **If you don't know, say so.** Research, ask. Don't pretend.

### Phase 4 — Fix and verify

1. **Create a failing reproduction.** Automated test if possible, one-off script in `tmp/` otherwise. Must fail now.
2. **Implement one fix.** Root cause only. No "while I'm here" refactors.
3. **Verify.** Reproduction passes. Nothing else broke.
4. **If three fixes have failed, stop.** This is an architecture problem, not a hypothesis problem. Question the pattern itself. Discuss with the user before a fourth attempt.

## Condition-based waiting (for flaky / timing bugs)

Never use `sleep(N)` as a fix. Poll for the actual condition with a bounded timeout:

```python
deadline = time.time() + timeout
while time.time() < deadline:
    if condition():
        break
    time.sleep(0.1)
else:
    raise TimeoutError(f"condition not met in {timeout}s")
```

A sleep that "seems long enough" is a bomb on a fuse.

## Defense in depth (after finding the root cause)

Once you've fixed the source, add validation at the next layer(s) so the same class of bug can't re-enter silently:
- Validate inputs at component boundaries
- Crash loudly on contract violations
- Log state transitions in long-running processes

Don't scatter defensive code everywhere. One or two layers past the actual fix is enough.

## Red flags — STOP, return to Phase 1

- "Quick fix for now, investigate later"
- "Just try changing X and see if it works"
- "It's probably X, let me fix that"
- "Pattern says X but I'll adapt it differently"
- Proposing solutions before tracing data flow
- "One more attempt" when 2+ have already failed

## When root cause is truly environmental

After systematic investigation, if the issue is timing, external, or truly non-deterministic:
- Document what you investigated
- Implement appropriate handling (retry, timeout with condition polling, loud error)
- Add logging / monitoring for future occurrences

But: 95% of "no root cause" verdicts are incomplete investigations. Double-check.

## Terminal state

Fix verified → if part of a task workflow, return to `up:verify`. Otherwise commit and move on.
