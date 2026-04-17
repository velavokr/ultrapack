# Ultrapack v1

**Status:** done
**Branch:** main
**Worktree:** none

## Design

Custom Claude Code skill pack, internal name `up`, repo/dir name `ultrapack`. Merges the load-bearing parts of [obra/superpowers](https://github.com/obra/superpowers) and Anthropic's [feature-dev](https://github.com/anthropics/claude-code/tree/main/plugins/feature-dev), drops the rest, freezes the behavior. Installed as a git repo symlinked into `~/.claude/plugins/`. No marketplace publishing.

Invocations always use `up:` prefix — `up:design`, `/up:make`, etc. — so the full pack surface is discoverable at a glance.

### Invariants

- Every task produces exactly one markdown file at `docs/tasks/<slug>.md`. No other layout.
- The task file is the source of truth for the task. Anything a later stage needs must be written there by an earlier stage.
- Review is never skipped, regardless of task size.
- Agents are dispatched with minimal context and never see the rationale behind changes they're reviewing. Reviewer independence must be preserved.
- Plans never contain complete code. Only surgical snippets for the most critical components.
- Plan steps always reference concrete locations: file path + line numbers + class/method/interface names.
- No skill, command, or agent in the pack hard-depends on anything outside the pack.
- The pack ships zero hooks. Users keep personal hooks in `~/.claude/hooks/`.

### Principles

1. **One task, one file.** `docs/tasks/<slug>.md` evolves through Design → Plan → Conclusion.
2. **Lean plans.** WHAT changes, not HOW. Snippets only for genuinely tricky code.
3. **Git-centered.** Branch per non-trivial task; worktrees for parallel work; commits are the timeline; task files live in the repo.
4. **Build around invariants.** Discovered in design, obeyed in plan, verified during review.
5. **Subagent independence.** Reviewers and researchers run in fresh contexts. No rationale leakage.
6. **TDD only where it fits.** Deterministic, reusable code that regressions would hurt. Not for training runs, EDA, one-offs, research.
7. **Manual testing is mandatory.** Agent must run what it built before claiming done. One-off snippets live in `tmp/`.
8. **ML hygiene.** Overfit one batch, check for data leaks, validate on a small run before scaling.
9. **Minimalistic.** Fewest words, fewest commands, fewest skills that still cover the workflow.
10. **Resume-ready.** Task file is comprehensive enough that a fresh-context agent can pick up where the last one stopped. Not required, but always possible.
11. **Fail fast, fail loud.** No silent fallbacks. No swallowed exceptions. Crash beats corrupt state.

### Structure

```
ultrapack/
├── plugin.json
├── README.md
├── skills/
│   ├── design/SKILL.md
│   ├── plan/SKILL.md
│   ├── execute/SKILL.md
│   ├── verify/SKILL.md
│   ├── review/SKILL.md
│   ├── document/SKILL.md
│   ├── debug/SKILL.md
│   ├── test-driven-development/SKILL.md
│   ├── data-engineering/SKILL.md
│   ├── ml-experiments/SKILL.md
│   └── git-worktrees/SKILL.md
├── commands/
│   ├── make.md
│   ├── try.md
│   ├── step-back.md
│   ├── handoff.md
│   └── reflect.md
└── agents/
    ├── explorer.md
    ├── reviewer.md
    └── researcher.md
```

### Task file template

```markdown
# <Task Title>

**Status:** design | planning | executing | reviewing | done
**Branch:** <git branch, or "main">
**Worktree:** <path, if used>

## Design
<purpose, scope, approach>

### Invariants
<specific things that must hold>

### Principles
<softer abstract guidance — concrete enough to check>

## Plan
<filled by up:plan>

## Conclusion
<filled by up:review — outcomes, deviations, future work with justification>
```

### Skills

All skill names use short verbs. Invocations: `up:<name>`.

- **design** — Brainstorm requirements and approach. Populate `## Design`, `### Invariants`, `### Principles` sections. Decide TDD yes/no with reason. Scope explicitly.
- **plan** — Expand Design into a lean Plan. Files + line numbers + class/method/interface names. WHAT changes per file. Test strategy. Order and dependencies. Open questions, risks, rollback. Code snippets only for critical components.
- **execute** — Implement the plan. Make incremental commits. Use `up:explorer` agent for codebase questions; use worktree if one was created. Update `## Plan` inline if the real world forces deviations (with a short note of why).
- **verify** — Build a checklist of what should and shouldn't hold. Run `/up:try`-style positive + negative checks on each. Manual smoke test. On failure: describe how each failure *should* have worked, loop back to execute. Out-of-scope items may be moved to Conclusion's Future Work only with a justification tied to a Design-scope line or a new fact discovered mid-execution. No slacking loophole. Does not write to the task file directly.
- **review** — Dispatch the `up:reviewer` agent. Fill `## Conclusion` with outcomes, deviations from plan, invariant adherence, future work (with justifications).
- **document** — Guidance for writing project docs, CLAUDE.md, READMEs, and in-code comments. Large classes/modules get a docstring explaining responsibilities. Inline comments reserved for genuinely tricky things (e.g. pinned library quirks). No aspirational sentences, no stale content, lists over tables.
- **debug** — Four-phase root-cause investigation (adapted from superpowers' systematic-debugging): reproduce, isolate, hypothesize, fix. No fixes without reproduction. Borrow the condition-based waiting and root-cause tracing sub-patterns.
- **test-driven-development** — RED → GREEN → REFACTOR. Applies only when: deterministic I/O contract AND called from more than one place AND regression would warrant a CI red light. Does NOT apply to: training runs, EDA, one-off scripts, research, UI tweaks where the test is "does it look right". TDD decision is recorded in Design.
- **data-engineering** — Forked from the user's existing `dataset-engineering` skill. Sequential stages, parallelism inside stages. Idempotent, atomic, retriable. Pre-flight checks. Write-to-temp-then-rename. Never nuke progress without asking.
- **ml-experiments** — Before committing compute: overfit one batch, small-run dynamics check, shape/dtype/stats on data loader. Data leaks: train/val/test isolation, no target leakage, no temporal leakage, no preprocessor fit on full data. Repro: seed, config, run ID. Monitoring live before launch. Never peek at test mid-experiment. Scale-up gated: small → medium → full.
- **git-worktrees** — Simplified fork of superpowers' `using-git-worktrees`. Smart directory selection (check `.worktrees/`, `worktrees/`, `CLAUDE.md` preference, then ask). Safety verification before writing.

### Commands

Invocations: `/up:<name>`.

- **make** — Orchestrates the full workflow. See Flow section below.
- **try** — Imported as-is from user's existing `~/.claude/commands/try.md`. Design one positive test case and one negative case; run both; report.
- **step-back** — Imported as-is. Circuit breaker: stop all current attempts, diagnose why approaches failed, propose a new direction.
- **handoff** — Imported and reshaped: if a task file is present, produce a session-summary into its Conclusion (or a scratch section). If no task file, write a general handoff summary. Stays general-purpose.
- **reflect** — Reflect on the current dialogue and extract learnings. Capture into CLAUDE.md, memory, or project docs as appropriate. Replaces the active behavior of the user's existing `/document` command.

### Agents

All three dispatched with minimal context. None see the full session history.

- **explorer** (Haiku 4.5). Tools: Glob, Grep, Read, Bash (readonly). Reads the *current codebase*. Traces execution paths, maps call graphs, returns 3–5 essential files with file:line references. No web, no writes.
- **reviewer** (Sonnet 4.6). Tools: Glob, Grep, Read, Bash (readonly). Single dispatch. Reads diff + plan + invariants from the task file, but never the rationale behind changes. First checks plan alignment (flags the plan if needed), then confidence-filtered code review (report only issues at confidence ≥ 80) with severity tiers (Critical / Important). Per issue: file:line, 1-line fix suggestion. Final verdict: merge-ready yes/no.
- **researcher** (Sonnet 4.6). Tools: WebSearch, WebFetch, Context7, Glob, Grep, Read, Bash (readonly). General-purpose. Decomposes a query and systematically investigates. Output shape depends on the query — not locked to SOTA/ML/literature. Examples: "what's the best X library for Y?", "how do other projects handle Z?", "what's SOTA in W for our project?".

### `/up:make` flow

1. **Slug.** Generate from task description, confirm with user.
2. **Task file.** Create `docs/tasks/<slug>.md` from template. No git branch yet.
3. **Design stage.** Run `up:design`. Populate Design, Invariants, Principles. Decide TDD yes/no.
4. **Branch / worktree decision.** Auto-infer from complexity: complex → branch + worktree; easy fix → current branch. Always confirm with user.
5. **Plan stage.** Run `up:plan`. Populate Plan.
6. **Execute stage.** Run `up:execute`. Incremental commits.
7. **Verify stage.** Run `up:verify`. Loop back to execute on failure until verify passes.
8. **Review stage.** Run `up:review`. Populate Conclusion.
9. **Finish.** Present merge/PR/cleanup options. Execute only after user chooses.

**Size-aware skips:**
- Trivial → skip Design and Plan (straight to Execute). Status file still created.
- Small → skip Design. Plan runs.
- Medium/large → full flow.
- Review is never skipped.

**Resume:** if `docs/tasks/<slug>.md` already exists, read Status and resume at the next stage.

### Distribution

- Public GitHub repo. Install via symlink into `~/.claude/plugins/` (or equivalent plugin location). No marketplace entry.
- `plugin.json` — name: `up`, version, description, author, license.
- `README.md` — install steps, philosophy in one screen, skill/command/agent index, pointers to `docs/tasks/` for the design history.

### Open questions

- None blocking. Refinements expected during implementation (e.g. exact reviewer prompt wording, verify's failure-proposal format).

## Plan

**Approach:** Doc-only pack. Every asset is a markdown file (SKILL.md, command.md, agent.md, plugin.json, README.md). No code to unit-test. Verification is install-and-invoke, not assertions. Order below is install-buildable: after each phase, the pack loads without error.

**Source references:**
- Superpowers skills at `superpowers/skills/<name>/SKILL.md`
- Feature-dev agents at `claude-code-plugins/plugins/feature-dev/agents/<name>.md`
- User's existing assets at `~/.claude/skills/<name>/SKILL.md`, `~/.claude/commands/<name>.md`

---

### Phase 1 — Scaffold

- **P1.1** `.claude-plugin/plugin.json` (edit — already exists with `name: "ultrapack"`)
  - Change `name` to `"up"` so invocations render as `up:design`, `/up:make`
  - Add `license`, `homepage`, `repository` (GitHub URL TBD on push), `keywords`
- **P1.2** `README.md` (create)
  - One-screen philosophy, install steps (symlink into `~/.claude/plugins/`), skill/command/agent index, link to `docs/tasks/ultrapack-v1.md`
- **P1.3** Top-level dirs: `skills/`, `commands/`, `agents/` (create empty)
- **P1.4** Commit: `feat: scaffold ultrapack plugin structure`

### Phase 2 — Import existing assets (zero adaptation)

- **P2.1** `commands/try.md` ← copy from `~/.claude/commands/try.md`
- **P2.2** `commands/step-back.md` ← copy from `~/.claude/commands/step-back.md`
- **P2.3** `commands/handoff.md` ← copy from `~/.claude/commands/handoff.md`. Reshape: if `docs/tasks/<slug>.md` detectable from session state, append handoff summary to its `## Conclusion`; otherwise stay general.
- **P2.4** `skills/data-engineering/SKILL.md` ← copy from `~/.claude/skills/dataset-engineering/SKILL.md`. Rename frontmatter `name: data-engineering`. Drop the `<system-reminder>` block ("user's #1 pain point") — that's personal, not pack-level. Keep everything else.
- **P2.5** Commit: `feat: import try, step-back, handoff, data-engineering`

### Phase 3 — Adapt superpowers skills (lean rewrites)

Each is a fresh short SKILL.md modeled on the source but trimmed to match principle 8 (minimalistic). Frontmatter: `name`, `description` (strong auto-trigger wording), optionally `model: haiku-4-5` where appropriate.

- **P3.1** `skills/design/SKILL.md` — adapted from `superpowers/skills/brainstorming/`
  - Drop: visual companion, design-doc file conventions (we write into task file instead), "hard gate" language (we're not implementation-blocking here)
  - Keep: one-question-at-a-time, 2-3 approaches with recommendation, incremental validation
  - Add: fill `## Design`, `### Invariants`, `### Principles` of the task file; record `TDD: yes/no (reason)`
- **P3.2** `skills/plan/SKILL.md` — adapted from `superpowers/skills/writing-plans/`
  - Drop: full-code requirement, TDD step-by-step format
  - Keep: file paths, no placeholders, scope check, self-review
  - Add: line numbers + class/method/interface names in file refs, invariants-referenced-by-plan, code snippets allowed only for critical components
- **P3.3** `skills/execute/SKILL.md` — adapted from `superpowers/skills/executing-plans/` + `subagent-driven-development/`
  - Drop: two-stage subagent review ceremony, separate executing-plans vs subagent-driven split
  - Keep: incremental commits, checkpoint per task
  - Add: `up:explorer` agent dispatch when codebase context is needed; deviation-from-plan notes inline in `## Plan`
- **P3.4** `skills/verify/SKILL.md` — adapted from `superpowers/skills/verification-before-completion/`
  - Drop: rigid command+exit-code output format
  - Keep: evidence-before-assertions, run commands freshly
  - Add: checklist construction (positive + negative), `/up:try`-style checks, manual smoke test, failure-loop-back-to-execute, Future Work justification rules
- **P3.5** `skills/review/SKILL.md` — adapted from `superpowers/skills/requesting-code-review/` + `receiving-code-review/` + superpowers code-reviewer agent concept
  - Drop: separate requesting vs receiving skills, confidence-score explanation verbosity
  - Keep: plan-alignment-first, severity tiers, file:line + fix suggestion
  - Add: single-dispatch to `up:reviewer` agent, fill `## Conclusion` with outcomes + deviations + future work
- **P3.6** `skills/debug/SKILL.md` — adapted from `superpowers/skills/systematic-debugging/`
  - Drop: optional sub-documents (condition-based-waiting, root-cause-tracing, defense-in-depth modules) — collapse best bits inline
  - Keep: no-fixes-without-reproduction, four-phase structure
  - Trim: target <150 lines
- **P3.7** `skills/test-driven-development/SKILL.md` — adapted from `superpowers/skills/test-driven-development/`
  - Keep: RED-GREEN-REFACTOR
  - Add: explicit applicability rule (3 conditions to use, 5 conditions to skip), TDD decision recorded in Design
- **P3.8** `skills/git-worktrees/SKILL.md` — adapted from `superpowers/skills/using-git-worktrees/`
  - Drop: smart directory selection verbosity (keep the actual rules, drop the prose)
  - Keep: safety verification, preference hierarchy (check `.worktrees/`, `worktrees/`, CLAUDE.md, then ask)
- **P3.9** Commit: `feat: adapt superpowers skills (design, plan, execute, verify, review, debug, tdd, git-worktrees)`

### Phase 4 — New skills

- **P4.1** `skills/document/SKILL.md` (new)
  - Scope: project docs, CLAUDE.md, READMEs, in-code comments
  - Rules: lead with why, no aspirational content, lists over tables, kill stale content
  - In-code: docstrings on large classes/modules explaining responsibilities; inline comments only for genuinely tricky things (pinned lib quirks, surprising invariants)
  - Auto-trigger: editing `.md` files, CLAUDE.md, docstrings
- **P4.2** `skills/ml-experiments/SKILL.md` (new)
  - Pre-compute: overfit one batch, small-run dynamics check, loader shape/dtype/stats
  - Leaks: train/val/test isolation, no target leakage, no temporal leakage, preprocessor fit only on train
  - Repro: seed, config, run ID, deterministic checkpoint paths
  - Monitoring: wandb/tensorboard live before launch, eval metrics defined up-front
  - Discipline: never peek at test mid-experiment, scale-up gated (small → medium → full)
  - Auto-trigger: touching `train.py`, training configs, model files
- **P4.3** Commit: `feat: add document and ml-experiments skills`

### Phase 5 — Agents

- **P5.1** `agents/explorer.md` — adapted from `feature-dev/agents/code-explorer.md`
  - Model: `haiku-4-5`
  - Tools: Glob, Grep, Read, Bash (readonly)
  - Output: execution trace + 3-5 essential files with file:line refs
  - Drop: parallel dispatch guidance, WebFetch/WebSearch, verbose output guidance
  - Trim aggressively for Haiku context budget
- **P5.2** `agents/reviewer.md` — synthesized from `feature-dev/agents/code-reviewer.md` + superpowers code-reviewer concept
  - Model: `sonnet-4-6`
  - Tools: Glob, Grep, Read
  - Single dispatch (not parallel multi-lens)
  - Flow: read task file's Plan + Invariants → plan-alignment check → code review with confidence ≥80 threshold → severity tiers Critical/Important → file:line + 1-line fix per issue → merge-ready verdict
  - Never sees rationale / session history
- **P5.3** `agents/researcher.md` (new, general-purpose)
  - Model: `sonnet-4-6`
  - Tools: WebSearch, WebFetch, Context7, Glob, Grep, Read, Bash (readonly)
  - Flow: decompose query → systematic investigation → synthesized answer
  - Output shape adapts to query — no locked format
- **P5.4** Commit: `feat: add explorer, reviewer, researcher agents`

### Phase 6 — Orchestrator command

- **P6.1** `commands/make.md` (new)
  - Flow (from spec): slug → task file → design → branch/worktree decision → plan → execute → verify-loop → review → finish
  - Size-aware skips: trivial skips design+plan; small skips design; review never skipped
  - Resume: if `docs/tasks/<slug>.md` exists, read Status, continue from next stage
  - Branch/worktree auto-inference: complex → branch+worktree, easy-fix → current branch, always confirm
- **P6.2** `commands/reflect.md` (new)
  - Reflect on current dialogue, extract learnings
  - Capture into CLAUDE.md, memory system, or project docs as appropriate
  - Replaces active behavior of user's existing `/document` command
- **P6.3** Commit: `feat: add make orchestrator and reflect commands`

### Phase 7 — Install and smoke test (verify phase)

- **P7.1** Symlink into `~/.claude/plugins/up` (or equivalent plugin install path)
- **P7.2** Fresh Claude Code session — confirm all 11 skills, 5 commands, 3 agents appear in system listing
- **P7.3** Invoke `/up:make "tiny throwaway task"` — verify flow starts, task file created, design stage runs
- **P7.4** Abort, clean up test task file
- **P7.5** Commit: `feat: v0.1.0 ready for install`

### Test strategy

No unit tests (doc-only pack). Verification via:
1. **Install smoke test** — pack loads, skills/commands/agents discoverable
2. **End-to-end dogfooding** — invoke `/up:make` on a real small task, confirm flow completes
3. **Independent review** — `up:reviewer` on the final diff, checking alignment to this plan

### Order & dependencies

- Phase 1 blocks everything (need scaffold)
- Phases 2-6 mostly independent; can be parallelized if using worktrees, but sequential is simpler
- Phase 6 depends on Phases 3-5 (make references skills + agents)
- Phase 7 depends on Phase 6

### Open questions / risks

- **Reviewer agent token budget** — feature-dev's reviewer runs on Sonnet in parallel with 3 lenses; single-dispatch on Sonnet 4.6 may need tightening to stay lean. Revisit during P5.2.
- **`/up:make` resume logic** — detecting the right task file when multiple exist. Probably: ask user if >1 in flight. Revisit during P6.1.
- **`up:` prefix behavior** — need to verify Claude Code's plugin namespacing actually renders as `up:design` / `/up:make`. May end up being just `design` / `/make` depending on how the harness handles it. Validate during P7.2.

### Rollback

Each phase is a single commit. `git revert` per phase to back out cleanly. Symlink in `~/.claude/plugins/` can be removed without affecting the repo.

## Conclusion

Outcome: ultrapack v1 is installed and loaded as the `up` plugin via `~/.claude/plugin-marketplace/up` (symlink to this repo). `/up:make`, `/up:reflect`, `/up:handoff`, `/up:try`, `/up:step-back` are discoverable. Skills (`design`, `plan`, `execute`, `verify`, `review`, `debug`, `test-driven-development`, `git-worktrees`, `document`, `data-engineering`, `ml-experiments`) and agents (`explorer` on Haiku 4.5, `reviewer` + `researcher` on Sonnet 4.6) load. Superpowers and feature-dev remain in the tree as read-only reference.

Plan adherence: mostly clean. Notable deviations, all recorded here rather than inline:
- Skill content was heavily rewritten mid-review on user feedback (reduced markdown, more pseudo-XML, informative section headers, two-roles review asymmetry, forbidden fallbacks in execute, plan-intro rewrite, scope-split moved to design).
- Handoff skill reworked from auto-append to draft-then-ask (append to current task Conclusion or create `docs/tasks/handoff-<slug>.md`).
- Review skill gained an "announce plan before editing" step.
- `make.md` initially got a per-stage docs-refresh, then corrected to run once after Review concludes the task.
- Design + plan skills picked up a backwards-compat check late in review.

Invariants: each verified by reading the final files:
- Agents dispatched without session history — reviewer + researcher + explorer prompts all instruct explicit working directory + task file path, no history pass-through (`agents/reviewer.md`, `skills/review/SKILL.md`).
- Task file is single source of truth — all six process skills read/write `docs/tasks/<slug>.md`, no sidecar plan/spec files.
- No auto-merge / auto-push — `/up:make` terminal step defers to user; `up:review` only presents options.
- `up:` namespacing verified post-install (`/reload-plugins` shows up:* skills/commands).

Review findings (dispatched `up:reviewer` at d86bd96..eebdcb6, merge-ready verdict, no Critical):
- Important #1 (Design's reviewer tool list missing `Bash (readonly)` vs actual `agents/reviewer.md`) — resolved, task file Design section updated.
- Important #2 (`commands/try.md` and `commands/step-back.md` missing YAML frontmatter description) — resolved, both files now have frontmatter matching the other commands.

Future work:
- Dogfood `/up:make` on a real feature task — Justification: `## Test strategy` explicitly named "end-to-end dogfooding" as the primary verification; ultrapack-v1 itself is the scaffolding task, not a representative feature.
- Revisit reviewer token budget after 3–5 real reviews — Justification: Open question from plan P5.2 ("feature-dev's reviewer runs on Sonnet in parallel with 3 lenses; single-dispatch on Sonnet 4.6 may need tightening"). Can't tune without empirical data.
- Reflect skill routing review — Justification: new command, untested in practice; its CLAUDE.md / memory / docs routing heuristics will need real-session data to calibrate.

Verified by: install smoke test (plugins load post-`/reload-plugins`), reviewer verdict (merge-ready after Important fixes), manual file reads to check every invariant claim above.
