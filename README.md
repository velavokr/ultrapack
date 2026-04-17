# ultrapack

A lean, opinionated Claude Code skill pack for spec-driven, git-centered development. Internal name: `up`. Invocations always use the `up:` prefix (`up:design`, `/up:make`, `up:reviewer`).

## Philosophy

- **One task, one file.** Every task produces a single markdown file at `docs/tasks/<slug>.md`, evolved through Design → Plan → Conclusion.
- **Lean plans.** WHAT changes, not HOW. Code snippets only for genuinely tricky parts.
- **Git-centered.** Branch per non-trivial task, worktrees for parallel work, commits are the timeline, task files live in the repo.
- **Invariants-first.** Discovered in design, obeyed in plan, verified during review.
- **Subagent independence.** Reviewers run in fresh contexts, never see change rationale.
- **TDD where it fits.** Deterministic, reusable code only. Not for training runs, EDA, one-offs, or research.
- **Manual testing mandatory.** Agent must run what it built before claiming done.
- **Minimalistic.** Fewest words, fewest commands, fewest skills that still cover the workflow.
- **Fail fast, fail loud.** No silent fallbacks.

## Install

```bash
git clone https://github.com/<your-user>/ultrapack ~/ultrapack
ln -s ~/ultrapack ~/.claude/plugins/up
```

Then restart Claude Code. Verify with `/up:make` or by listing available skills.

## Contents

### Skills (11)

- `up:design` — Brainstorm requirements and approach; populate Design + Invariants + Principles.
- `up:plan` — Lean plan: files, line numbers, class/method/interface names, invariants, test strategy.
- `up:execute` — Implement the plan, incremental commits, dispatch `up:explorer` when needed.
- `up:verify` — Checklist of what should and shouldn't hold, smoke test, loop back on failure.
- `up:review` — Dispatch `up:reviewer`, fill Conclusion with outcomes and deviations.
- `up:document` — Guidance for docs, CLAUDE.md, READMEs, and in-code comments.
- `up:debug` — Four-phase root-cause investigation; no fixes without reproduction.
- `up:test-driven-development` — RED → GREEN → REFACTOR, only when the task qualifies.
- `up:data-engineering` — Sequential stages, idempotent/atomic/retriable, pre-flight checks.
- `up:ml-experiments` — Overfit one batch, check for leaks, scale-up gated.
- `up:git-worktrees` — Smart directory selection, safety verification.

### Commands (5)

- `/up:make` — Orchestrate the full flow: slug → task file → design → branch → plan → execute → verify → review.
- `/up:try` — One positive + one negative check; run and report.
- `/up:step-back` — Circuit breaker: stop, diagnose why approaches failed, propose new direction.
- `/up:handoff` — Session-summary for continuation.
- `/up:reflect` — Reflect on dialogue, extract learnings into CLAUDE.md / memory / docs.

### Agents (3)

- `up:explorer` (Haiku 4.5) — Codebase tracing, file:line refs, essential-files list.
- `up:reviewer` (Sonnet 4.6) — Single-dispatch, plan-alignment-first, confidence-filtered, severity-tiered.
- `up:researcher` (Sonnet 4.6) — General-purpose: decompose query and systematically investigate.

## Design history

See `docs/tasks/ultrapack-v1.md` for the full design spec and implementation plan.

## License

MIT
