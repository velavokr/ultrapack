# Interface-first planning and parallel execution

**Status:** planning
**Branch:** interface-first-parallel
**Worktree:** .worktrees/interface-first-parallel
**Mode:** hands-off

## Design

Problem. `### Execution batches` (landed in `parallel-phase-exec`) parallelizes by file-path disjointness, but planners still write coarse, coupled phases, so most batches collapse to near-sequential. Theory: forcing interface-first planning atomizes work at the contract boundary; the filesystem-disjointness check becomes a cheap lower-level guard instead of the dispatch primitive.

Approach. `## Plan` gains two subsections — `### Interfaces` (contract definitions) and `### Interface graph` (the execution shape, dataflow grammar). Each phase declares what it produces, what it consumes, and which files it owns. `up:uexecute` topo-sorts the graph into waves; within a wave, phases dispatch in parallel bounded by `@`-disjointness. Two new post-dispatch checks: Boundary (diff ⊆ `Owns`) per returned phase, Wiring (callers of each IF match the declared signature) at end of graph. `up:uverify` adds one CK per IF. Replaces `### Execution batches` (grammar removed, not layered).

Plan subsections:

```
### Interfaces
- IF1 — `Parser.parse(s: str) -> AST` — canonical AST; raises on syntax error
- IF2 — `Runner.run(ast: AST) -> Result` — pure; no I/O

### Interface graph
- PH1  -> IF1             @ parser.py, ast.py
- PH2  -> IF2             @ runner.py
- PH3  IF1, IF2 ->        @ main.py
```

Grammar. `PH<N>  <consumes> -> <produces>   @ <owns>`. Empty left = source; empty right = sink/wiring. Comma-separated IF lists; comma-separated paths after `@`. Single-phase plans omit both subsections.

Execution. Derive waves via topological sort over `->`. Within a wave, check `@` disjointness; dispatch each phase as a concurrent `up:implementer` (`commit: defer`); dispatcher commits each returned phase in PH-order; run Boundary + plan-diff + consistency per commit. After the final wave commits, run Wiring check across the whole graph. Falls back to serial one-phase-at-a-time when `### Interface graph` is absent (IV6).

Boundary check. For each returned phase, `git show <sha>` — every changed path must be in the phase's `Owns`. Trespass = halt, log deviation, re-dispatch with tightened scope or escalate to `up:uplan`.

Wiring check. For each IF, `grep` callers; verify each call site matches the declared signature (name, arity, argument types if expressible). For doc-only targets (SKILL.md edits in this repo's own meta-tasks), "signature" degenerates to the declared section anchor / public contract and wiring check becomes an anchor-presence grep.

Verify addition. Checklist grows a CK per IF: "IF<n> honored" — contract asserted end-to-end (real call, real return, or structural match).

Backwards-compat. `### Execution batches` grammar removed. No in-flight tasks use it (all `docs/tasks/*.md` are `done`). External users' existing plans with old grammar will be ignored by the new executor; plans without `### Interface graph` fall back to serial. Hard break, no shim — PC4.

TDD: no (reason: doc-only plugin change — SKILL.md and agent markdown; no runtime code to test).

### Invariants
- IV1 — `## Plan` declares every cross-phase contract as `IF<n>` with signature + contract sentence.
- IV2 — Every phase in `### Interface graph` declares `@ <paths>`; `<consumes> ->` and `-> <produces>` are declared when relevant.
- IV3 — Boundary check runs after each phase's commit: diff ⊆ phase's `Owns` or halt (no silent widening).
- IV4 — Wiring check runs once, after the last wave's phases commit: every IF has a caller match (or an anchor-presence match for doc targets).
- IV5 — Verify checklist has ≥1 CK per IF.
- IV6 — Plans without `### Interface graph` execute serially, one implementer per phase — no behavior change for single-phase or legacy plans.
- IV7 — One commit per phase, regardless of wave or parallelism (carried over from `parallel-phase-exec`).

### Principles
- PC1 — Interfaces are plan-text. Code stubs (a PH0 scaffolding phase) are optional and decided per task; not required by the skill.
- PC2 — `@` (owned files) is the filesystem-parallelism boundary; IFs are the logical-coupling boundary. Both must be declared; neither subsumes the other.
- PC3 — Fail loud on boundary and wiring violations. No silent re-dispatch, no silent scope expansion.
- PC4 — Replace, don't layer. `### Execution batches` grammar is removed; IFs + `@` are the single source of truth for dispatch.
- PC5 — Waves are derived from the graph, never hand-declared. The planner writes the graph; the executor infers the waves.

### Assumptions
- AS1 — Interface-first planning yields strictly more parallel dispatch than file-path batching (the task's load-bearing hypothesis; conclusion must report whether observed).
- AS2 — Planners can nail interface signatures before seeing implementations (standard top-down design assumption).
- AS3 — `git show <sha>` against the phase's declared `@` paths is sufficient for Boundary; implementers don't silently touch files outside their declared scope.
- AS4 — `grep` of caller sites + signature string-match is sufficient for Wiring for most tasks; tasks needing stronger checks can escalate to an integration-smoke phase.
- AS5 — For doc-only tasks, "interface" reduces to SKILL.md section anchors / declared public contracts; the same grammar applies without strain.
- AS6 — Removing `### Execution batches` strands no in-flight task in this repo (all current task files are `done`).

### Unknowns
- UK1 — Plan-text interfaces vs a dedicated PH0 stub phase that lays down empty types/protocols before the parallel wave. Current lean: plan-text only; stubs are an optional pattern the planner can choose per task.
- UK2 — Wiring-check mechanism for code targets — structural grep vs a tiny integration smoke vs a dedicated terminal phase. Current lean: structural grep; planner can escalate to a terminal wiring phase (a sink node) when the integration is non-obvious.
- UK3 — Boundary-check strictness — strict file-scope (any diff outside `Owns` halts) vs semantic (allow formatting-only touches to neighbors). Current lean: strict file-scope, rg-checkable.
- UK4 — When an IF is discovered wrong mid-execute (consumers coded against a bad signature): re-dispatch all consumers, or invoke `up:uplan` to rewrite the graph. Plan/execute will decide.
- UK5 — Whether the wave scheduler needs an explicit "max parallelism" cap (e.g. don't dispatch >N implementers at once). Current lean: no cap; surface if we hit practical limits.

## Plan
<empty — filled by up:uplan>

## Verify
<empty — filled by up:uverify>

## Conclusion
<empty — filled by up:ureview>

### Hands-off decisions
- make: size=Medium — multi-skill restructure (uplan, uexecute, uverify); hands-off default.

### Deferred (needs user input)
<empty>

## Seed (from /up:make args)

Prior work added batch execution and parallelization, but execution still trends sequential. Proposed shift:

1. Plan defines interfaces. As many components as possible should be developable knowing only the interfaces.
2. Phases build components that implement those interfaces.
3. Executor dispatches as many implementers in parallel as possible.
4. Executor checks each implementer didn't alter internals of components outside its assigned scope.
5. Executor checks components wire together as intended after implementers finish.
6. Verify also checks interfaces.

Hypothesis: explicit interface separation and independent execution improves both planning and execution quality, for free.
