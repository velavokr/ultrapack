---
name: design
description: Use before any creative work — creating features, building components, adding functionality, modifying behavior. Explores intent, scope, invariants, and principles before anything is planned or written.
---

# Design

Turn an idea into a validated spec through collaborative dialogue. Output: the `## Design` section of the task file at `docs/tasks/<slug>.md`, with `### Invariants` and `### Principles` subsections populated and a TDD decision recorded.

## When to use

- User describes a feature, a change, or a problem to solve
- Any time you're tempted to start coding without a shared picture of what you're building
- Even for "simple" tasks: five minutes of design saves hours of re-work

## Process

1. **Explore project context.** Files, recent commits, existing patterns. Do not skip this.
2. **Scope check.** If the ask spans multiple independent subsystems, flag it. Help decompose into sub-tasks, each with its own task file.
3. **Ask clarifying questions, one at a time.** Prefer multiple choice. Focus on purpose, constraints, success criteria.
4. **Propose 2–3 approaches.** Always lead with a recommendation and its reasoning. Don't hedge.
5. **Present the design in sections.** Scale each to its complexity. Ask for approval section by section.
6. **Identify invariants.** Specific things that must hold. "The `Dataset` class must not import from `training/`."
7. **Identify principles.** Softer abstract guidance, concrete enough to check. "Fail fast, no silent fallbacks." "Prefer composition over inheritance." Not vague: "prefer composition" alone is too abstract.
8. **Decide TDD.** Use `up:test-driven-development`'s applicability rule. Record `TDD: yes` or `TDD: no (reason)` in Design.
9. **Write to task file.** Fill `## Design`, `### Invariants`, `### Principles`. Self-review for placeholders, contradictions, scope, ambiguity. Fix inline.
10. **User reviews the task file.** Wait for approval before invoking `up:plan`.

## Task-file output shape

```markdown
## Design
<purpose, scope, chosen approach, key decisions, TDD: yes/no (reason)>

### Invariants
- <specific thing that must hold>
- <...>

### Principles
- <softer guidance — concrete enough to check>
- <...>
```

## Key rules

- **One question per message.** No batching.
- **YAGNI ruthlessly.** Cut anything not needed for the stated goal.
- **Explore alternatives.** Propose 2–3 approaches before committing.
- **Follow existing patterns.** Don't unilaterally restructure. Include targeted improvements only if they serve this task's goal.
- **Isolation and clarity.** Units with one clear purpose, well-defined interfaces, understandable in isolation.

## After design

Design's terminal state is invoking `up:plan`. Do not invoke any other skill. Do not write code.
