---
name: review
description: Use after verify passes to get an independent check. Dispatches up:reviewer agent, processes feedback rigorously (no performative agreement), fills the task file's Conclusion.
---

# Review

Review is a process, not a section. It produces the `## Conclusion` section of the task file.

## When to review

- After `up:verify` passes
- Before merge to main
- Before opening a PR
- Review is **never skipped**, regardless of task size

## Process

### 1. Dispatch up:reviewer

Get git SHAs:
```bash
BASE_SHA=$(git rev-parse HEAD~<N>)  # N = number of commits in this task's work
HEAD_SHA=$(git rev-parse HEAD)
```

Dispatch the `up:reviewer` agent with:
- Task file path (`docs/tasks/<slug>.md`)
- `BASE_SHA` and `HEAD_SHA`
- **Do not pass session history.** The reviewer must not see your rationale — only the Plan, Invariants, and diff.

### 2. Read feedback, don't react

Receive the reviewer's output without immediate reply:
- **Critical**: fix before proceeding
- **Important**: fix before merge
- **Suggestions** (if any): note, don't necessarily implement

### 3. Evaluate each item

For every finding:
1. **Restate** the concern in your own words. If you can't, ask for clarification.
2. **Verify** against the codebase. Does the issue actually exist as described?
3. **Evaluate** technically: is the suggested direction correct for *this* codebase?
4. **Decide**: implement, push back (with reasoning), or escalate to user.

### 4. Apply fixes

Fix Critical and Important issues. Commit each as a separate logical unit. Re-dispatch the reviewer if the changes are substantial.

### 5. Write the Conclusion

Fill `## Conclusion` in the task file:

```markdown
## Conclusion

**Outcome:** <what works now, in one paragraph>

**Plan adherence:** <deviations from the plan and why>

**Invariants:** <each invariant and how it was verified>

**Review findings:**
- Critical: <resolved>
- Important: <resolved or explicitly deferred>

**Future work:**
- <item> — Justification: <Design-scope line excluded this> OR <new fact discovered during execution>
- (No unjustified items allowed — this is not a dumping ground for incomplete in-scope work.)

**Verified by:** <smoke tests run, manual checks, reviewer>
```

## Receiving feedback — rules

**Never:**
- "You're absolutely right" / "Great catch" / "Thanks"
- Implement blindly without verifying against the codebase
- Batch fixes without testing each
- Partial implementation when items may be related — clarify all first

**Do:**
- Verify against the codebase reality
- Push back with technical reasoning when wrong
- Ask for clarification when unclear
- Fix → show the diff → move on (actions > words)

**Pushback is legitimate when:**
- Suggestion breaks existing behavior
- Reviewer lacks full context that only you have
- Suggestion violates YAGNI (unused feature being "properly implemented")
- Conflicts with explicit Design/Invariants decisions

## Never

- Accept "ready to merge" without evidence
- Merge with open Critical or Important findings
- Skip the Conclusion write-up
- Run review on yourself (use the subagent, preserve independence)

## Terminal state

Conclusion written, all Critical/Important resolved → present merge/PR/cleanup options. The user chooses; you don't auto-merge.
