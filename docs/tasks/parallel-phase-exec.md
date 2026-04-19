# Parallel Phase Execution

**Status:** executing
**Branch:** parallel-phase-exec
**Worktree:** .worktrees/parallel-phase-exec
**Mode:** hands-off

## Design

Goal: let `up:uexecute` dispatch multiple `up:implementer` agents in parallel when phases don't depend on each other, and let one implementer carry multiple small sequentially-dependent phases. Keeps the current serial behavior as the default so existing plans work unchanged.

Approach (opt-in via the Plan). `up:uplan` gains an optional `### Execution batches` subsection that declares how phases are grouped for dispatch. `up:uexecute` reads that subsection and dispatches each batch; within a batch, each group ("bundle") runs in one implementer and groups run in parallel via multiple `Agent` tool calls in one response. When the subsection is absent, execute falls back to today's one-phase-per-implementer serial loop.

Batch declaration format in `## Plan`:

```
### Execution batches
- B1: [PH1] || [PH2 → PH3]    # two bundles in parallel; bundle B does PH2 then PH3
- B2: [PH4]                    # solo phase
- B3: [PH5] || [PH6]
```

- `||` separates bundles that run in parallel within the batch.
- `→` inside a bundle means the implementer runs those phases sequentially in one dispatch, commits each.
- Bundling criterion: PH N+1 depends on PH N (imports a symbol, edits what N just created) AND the pair together is still small (a handful of files, low risk, no structural divergence from Plan).

Disjointness invariant. Phases in different bundles of the same batch must touch disjoint file paths (exact paths, not directories). `up:uplan` declares files per phase already (File structure section). `up:uexecute` verifies disjointness from the Plan before dispatching the batch; on overlap, stops and logs under Deferred. Bundled phases in the same bundle may overlap freely (same implementer, sequential).

Commits in parallel batches. The dispatcher serializes git. In a parallel batch, each implementer does edits + tests + self-review but does NOT commit. It reports the changed files and the intended commit message. The dispatcher commits each returned phase sequentially (one commit per phase, order: B1's bundles alphabetically or by lowest-PH-number), runs plan-diff check and consistency pass after each commit, then moves to B2. Solo and bundled-single-implementer phases commit as today (implementer commits). Rationale: git index and HEAD are process-global in one worktree; parallel `git add`/`git commit` races are the failure mode we're avoiding. Serializing only the commit step keeps code edits parallel.

Failure handling in a parallel batch. If any implementer in the batch returns `BLOCKED` or `NEEDS_CONTEXT`, wait for all siblings in the batch to complete, commit the ones that returned `DONE`, then handle the failure(s) before moving to the next batch. Do not abort siblings mid-work (their results are still useful and their commits are reversible). If an implementer returns `DONE_WITH_CONCERNS`, commit and surface the concern per the existing rule.

Changes required:

- `plugins/up/skills/uplan/SKILL.md` — add optional `### Execution batches` subsection to the Format block; add a short "When to declare batches" paragraph (when to parallelize, when to bundle, disjoint-files rule); leave plans without it working as today.
- `plugins/up/skills/uexecute/SKILL.md` — replace the short "Parallel dispatch" paragraph with a dedicated "Batched dispatch" section: read `### Execution batches`, verify file disjointness, dispatch one or more implementers per batch (parallel Agent calls in a single response for parallel bundles), handle the serialized-commit protocol, run plan-diff + consistency after each phase, proceed to next batch.
- `plugins/up/agents/implementer.md` — add a "commit mode" to the What-you-receive list: `commit: self` (default, today's behavior) vs `commit: defer` (parallel batch — stage/summarize only, dispatcher commits). In `commit: defer` mode, the Report Format includes `Commit message (proposed): <message>` and omits `Commit: <sha>`.

TDD: no (reason: doc-only plugin change — SKILL.md and agent markdown; no runtime code to test)

### Invariants
- IV1 — One commit per phase, regardless of execution mode (preserves rollback surface).
- IV2 — Phases in different bundles of the same batch must touch disjoint file paths; dispatcher verifies from the Plan before dispatching the batch.
- IV3 — Plan-diff check and consistency pass run after each phase's commit, in every mode (serial, parallel, bundled).
- IV4 — The Plan is the contract for parallelism: batching is declared in `### Execution batches`, never inferred by the dispatcher at runtime.
- IV5 — Plans with no `### Execution batches` subsection execute exactly as today (serial, one implementer per phase). No behavior change by default.

### Principles
- PC1 — Opt-in parallelism. Absence of a batch declaration means serial execution.
- PC2 — Serialize git, parallelize edits. Code changes can run concurrently; commits do not.
- PC3 — Keep the implementer contract minimal. The only new thing it learns is "commit: self | defer".
- PC4 — Fail loud on disjointness violations. No silent fallback to serial when files overlap — stop and log under Deferred.

### Assumptions
- AS1 — Claude Code's Agent tool supports multiple concurrent invocations in a single response (the tool docs state this explicitly).
- AS2 — Parallel implementers running Edit/Write/Read in the same working directory on disjoint file paths do not race at the filesystem layer.
- AS3 — Parallel implementers do not invoke git write operations themselves in `commit: defer` mode, so only the dispatcher touches the git index/HEAD.
- AS4 — The existing Plan's per-phase File structure listing is reliable enough for disjointness checking; implementers don't silently touch files outside their phase's list.

### Unknowns
- UK1 — Exact grammar for `### Execution batches`: the `[PH1] || [PH2 → PH3]` shape proposed here is one option; plan stage may propose a cleaner alternative (e.g. nested list) if the inline grammar is hard to read for bigger batches.
- UK2 — Whether the consistency pass in a parallel batch needs to wait for ALL bundles in the batch to commit before running (so it sees the whole batch's diff), or can run per-phase as phases commit. Current lean: per-phase, matching today's behavior; revisit if it surfaces false-negatives.
- UK3 — Resolved: `up:uplan` emits `### Execution batches` only when real parallelism exists; single-phase / obviously-sequential plans omit it. Planner owns ordering and batch composition.

## Plan

Approach: three independent doc edits — implementer agent gets a commit-mode input, uplan gains an optional batch-declaration subsection, uexecute gains a batched-dispatch section that ties them together. PH1 and PH2 touch disjoint files and have no dependency; PH3 references both conceptually (cross-links in prose) and lands last.

### PH1 — Implementer: commit mode

- **1.1** `plugins/up/agents/implementer.md:11-17` (modify) — "What you receive"
  - Add bullet: `Commit mode: self | defer` — `self` = implementer commits (today's behavior, default). `defer` = dispatcher commits; implementer stages + tests + reports intended message.
  - Respects: IV1, PC3
- **1.2** `plugins/up/agents/implementer.md:21-28` (modify) — "Process"
  - Step 6 "Commit" becomes conditional: if `commit: self`, commit as today; if `commit: defer`, stage the changed files (`git add <paths>`) and skip the commit.
- **1.3** `plugins/up/agents/implementer.md:48-68` (modify) — "Report Format"
  - In `commit: self` mode, report carries `Commit: <sha> <message>` as today.
  - In `commit: defer` mode, replace the `Commit:` line with `Commit message (proposed): <one-line message>` and add `Staged files: <path>, <path>`.
- **1.4** `plugins/up/agents/implementer.md:40-46` (modify) — "Forbidden"
  - Add: in `commit: defer` mode, do not run `git commit`, `git reset`, or branch/tag ops; staging only.
- Commit: `feat(implementer): add commit: self|defer mode for parallel dispatch`

### PH2 — Uplan: Execution batches subsection

- **2.1** `plugins/up/skills/uplan/SKILL.md:73-80` (modify) — "Required when relevant" list
  - Add: `Execution batches: declare parallel groups and multi-phase bundles when real parallelism exists. Omit otherwise.`
  - Respects: IV4, IV5, PC1
- **2.2** `plugins/up/skills/uplan/SKILL.md:82-108` (modify) — Format block
  - Append `### Execution batches (optional — omit when all phases run sequentially and alone)` with the grammar: `- B<N>: [PH<i>] || [PH<j> → PH<k>]` — `||` separates bundles that run in parallel, `→` means one implementer runs those phases sequentially.
- **2.3** `plugins/up/skills/uplan/SKILL.md:108-118` (modify) — new subsection "When to declare batches" (insert after Format block, before Self-review)
  - Parallelize phases only when their File structure bullets declare disjoint file paths.
  - Bundle PH N+1 with PH N when they have a tight dependency AND the pair is small (few files, low risk).
  - Omit the subsection when no real parallelism or bundling exists. Do not invent batches to fill space.
  - Respects: IV2, PC1, PC4
- **2.4** `plugins/up/skills/uplan/SKILL.md:110-118` (modify) — Self-review
  - Add item: "If `### Execution batches` is present, every bundle's phases have disjoint file paths from every other bundle in the same batch; bundled phases have a declared dependency."
- Commit: `feat(uplan): add optional Execution batches subsection`

### PH3 — Uexecute: batched dispatch

- **3.1** `plugins/up/skills/uexecute/SKILL.md:82-85` (modify) — replace the short "Parallel dispatch" paragraph in the "Dispatch per phase" section with a pointer to the new section below.
  - Respects: IV4, IV5
- **3.2** `plugins/up/skills/uexecute/SKILL.md:40-58` (modify) — "Per-phase loop"
  - Reframe as "Per-batch loop". If the Plan has `### Execution batches`, iterate batches; else fall back to today's one-phase-at-a-time loop (IV5).
  - Within a batch: verify file disjointness across bundles from the Plan's File structure; on overlap, stop and log under `### Deferred (needs user input)` (PC4).
- **3.3** `plugins/up/skills/uexecute/SKILL.md` (insert new section, between current "Dispatch per phase" and "TDD" — ~line 86) — "Batched dispatch"
  - Reading the batch declaration; mapping bundles to implementer dispatches; using multiple `Agent` tool calls in one response for parallel bundles; `commit: defer` for parallel bundles, `commit: self` for solo bundles and for multi-phase bundles (one implementer, sequential commits).
  - Serialized commit protocol: await all bundles in the batch, then for each returned `DONE`/`DONE_WITH_CONCERNS` report in order of lowest PH number, run `git add <staged paths>` (if not already), `git commit -m "<proposed message>"`, then plan-diff check (IV3) and consistency pass (IV3) for that phase.
  - Failure handling: on `BLOCKED` or `NEEDS_CONTEXT` from one bundle, wait for siblings, commit siblings' successful results, then diagnose the failure (re-dispatch, uplan, or stop). Do not abort siblings mid-work.
  - Respects: IV1, IV2, IV3, PC2, PC4
- **3.4** `plugins/up/skills/uexecute/SKILL.md:83-85` (modify) — "Dispatch per phase → Pass in the dispatch prompt"
  - Add bullet: `Commit mode: self | defer` — default `self`, set `defer` only when this phase is in a parallel bundle per `### Execution batches`.
- **3.5** `plugins/up/skills/uexecute/SKILL.md:56-57` (modify) — "Do not batch phases" sentence
  - Replace with: "Batch only per `### Execution batches` in the Plan; never infer batches at runtime (IV4)."
- Commit: `feat(uexecute): batched dispatch with serialized commits`

### Order & dependencies

- PH1 and PH2 touch disjoint files (`implementer.md` vs `uplan/SKILL.md`) with no import/data dependency.
- PH3 touches `uexecute/SKILL.md` only, but its prose references the contracts introduced by PH1 (commit mode) and PH2 (batch subsection), so it lands after them.

### Execution batches

- B1: [PH1] || [PH2]
- B2: [PH3]

### Risks / rollback

- RK1 — Grammar in `### Execution batches` (`||`, `→`) may be unclear to a reader at first glance; mitigated by a one-line legend in the Format block and a short "When to declare batches" section. Rollback: single commit revert on PH2.
- RK2 — Serialized-commit protocol in parallel mode could drop an implementer's staged changes if the dispatcher crashes between `return` and `git commit`; mitigated by the dispatcher running `git add` + `git commit` back-to-back with no other operations in between. Rollback: single commit revert on PH3.

## Verify
<empty — filled by up:uverify>

## Conclusion
<empty — filled by up:ureview>

### Hands-off decisions
- size: Medium — default in hands-off; touches up:uexecute skill and possibly up:implementer agent, needs full design + plan
- make: dedicated branch + worktree at .worktrees/parallel-phase-exec — hands-off safest-reversible default
- udesign: UK3 resolved by user (only emit subsection when parallelism exists); UK1/UK2 left for planner
- uplan: plan auto-approved (hands-off)

### Deferred (needs user input)
