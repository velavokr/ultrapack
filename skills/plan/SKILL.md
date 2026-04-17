---
name: plan
description: Use after design to turn a validated spec into a lean implementation plan. Outputs the `## Plan` section of the task file — files, line numbers, class/method names, invariants, test strategy, order.
---

# Plan

Plans describe **WHAT changes, not HOW**. Code appears only for genuinely tricky components. Target: the reader can estimate effort and spot missing work in under two minutes.

## Output

Fill the `## Plan` section of `docs/tasks/<slug>.md`. Do not create a separate plan file.

## Scope check

If the spec spans multiple independent subsystems, stop. Split into separate tasks, each with its own task file. A plan should produce working, testable work on its own.

## Plan contents

Required for every plan:
- **Approach** — 2-3 sentences on the strategy and why it fits
- **File structure** — for each file to create/modify: path + line numbers + class/method/interface names affected
- **Per-file bullets** — what changes, not how. As many bullets as needed.
- **New/changed interfaces** — signatures only, no bodies
- **Invariants referenced** — which invariants from Design each change preserves
- **Test strategy** — behaviors to cover, not test code. Decide per task whether TDD applies (see `up:test-driven-development`).
- **Order + dependencies** — phases/tasks, what blocks what
- **Open questions / risks / rollback** — what could go wrong, how to back out

Optional:
- **Code snippets** — only for the most critical component (a regex, an algorithm, a data model, a non-obvious API call). If you're tempted to include full code, you're over-planning.

## Format

```markdown
## Plan

**Approach:** <2-3 sentences>

### Phase 1 — <name>

- **N.1** `path/to/file.ext:lineA-lineB` (create|modify)
  - `ClassName.method_name()` — <what changes>
  - <additional bullets>
  - Invariant: <which invariant from Design this respects>
- **N.2** ...
- Commit: `<message>`

### Phase 2 — <name>
...

### Test strategy
<behavior list, per file or per phase>

### Order & dependencies
<what blocks what, parallelizable phases>

### Open questions / risks / rollback
- <item>
```

## Self-review (run inline)

1. **Spec coverage** — every Design/Invariants/Principles item maps to at least one plan bullet
2. **Placeholder scan** — no "TBD", "handle edge cases", "add validation", "similar to task N", "write tests for the above"
3. **Consistency** — method/class names match across bullets; interfaces in later tasks reference what earlier tasks define
4. **Leanness** — if plan exceeds ~1 screen per 1-2 day of work, split the task or trim
5. **Invariants** — each invariant has a referencing bullet somewhere

Fix issues inline. No re-review.

## What NOT to put in a plan

- Full implementation code
- Full test code
- Line-by-line pseudo-code narration
- Speculative refactors outside the task's scope
- Duplicate content ("similar to task N, but with X")

## Terminal state

After plan is written and self-reviewed: ask the user to review. On approval, invoke `up:execute`. Do not invoke any other skill.
