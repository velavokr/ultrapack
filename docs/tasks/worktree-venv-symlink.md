# Worktree environment sharing

**Status:** done
**Branch:** worktree-venv-symlink
**Worktree:** .worktrees/worktree-venv-symlink
**Mode:** hands-off

## Design

A fresh worktree has no project environment — no installed dependencies, no language runtime pointer. Tools that resolve through ancestor walks (ty, ruff-lsp, pyright, node's resolver, etc.) either emit bogus env-path diagnostics or reinstall from scratch. Both are avoidable, because at creation time the parent repo's environment is identical to what the worktree needs (same commit base, same manifest).

General principle to add to `up:git-worktrees`: at creation, share the main repo's environment into the worktree. Rebuild locally only if deps later diverge. The skill states the principle language-agnostically and gives Python as a worked example; for any other stack, the reader does the analogous thing.

Worked example — Python (uv / venv):
- If the main repo has `.venv/`, symlink it into the worktree at `<worktree>/.venv` using an absolute path.
- If it doesn't, run the project's install command (`uv sync`, `pip install -e .`, etc.) in the worktree.
- Skip entirely if the worktree already has `.venv` (real or symlink).

Extrapolation guidance (in the skill): for any stack with a per-project environment directory, do the analogous share-from-main. Examples the reader can infer: Node `node_modules`, Ruby `vendor/bundle`, Rust `target`. Go and Gradle share caches globally (`$GOMODCACHE`, `~/.gradle`), so no per-worktree action is needed. The skill does not enumerate an exhaustive table — the principle + one example carries it.

Follow-up note in the skill: if the worktree changes dependencies (edits the manifest / lockfile), replace the shared environment with a local install. The skill doesn't detect this — it's guidance for whoever edits deps.

Absolute symlink target (`$(git -C <main-repo-root> rev-parse --show-toplevel)/<env-dir>`) so nested worktree paths don't need a relative-dot count.

Rejected alternatives:
- Exhaustive ecosystem table in the skill — brittle and pretends completeness; the principle + analogy is the point.
- Per-ecosystem pinning config (e.g. `[tool.ty.environment] python = ".venv/bin/python"`) — still requires the env to exist in the worktree; the shared env is the load-bearing fix.
- Auto-detecting dep-divergence (manifest/lockfile hash compare main vs worktree) — at create-time they're always identical; over-engineered.

TDD: no (doc-only skill edit; verification is install-and-invoke).

### Invariants
- IV1 — Shared-environment symlink uses an absolute path to the main repo's env dir, not a relative path.
- IV2 — Never overwrite an existing environment (real or symlink) in the worktree.

### Principles
- PC1 — State the general share-from-main principle; give one worked example (Python); let the reader extrapolate to other stacks.
- PC2 — Fail loud if a fallback install command is attempted and fails; no silent fallback to a broken env.

### Assumptions
- AS1 — A worktree of the same repo uses the same interpreter/runtime versions as the main repo (same manifest pins).
- AS2 — At creation time, worktree deps match main (fresh branch); symlink is always safe at this moment.
- AS3 — Users/agents that later change deps in the worktree will replace the symlink with a local install per the skill's follow-up note.

## Plan

Approach: one edit to `plugins/up/skills/git-worktrees/SKILL.md` — insert a new "Share environment from main" section before "Baseline setup", stating the general principle, giving Python as the worked example, and noting the follow-up rule when deps diverge. Tighten baseline-setup's Python line so a shared `.venv` is respected.

### PH1 — Document environment sharing in git-worktrees

- **1.1** `plugins/up/skills/git-worktrees/SKILL.md` (modify) — insert a new `## Share environment from main` section between `## Creation` and `## Baseline setup`:
  - Principle: share the main repo's environment at creation; rebuild locally only if deps later diverge.
  - Python worked example: absolute-path symlink of `<main>/.venv` → `<worktree>/.venv` when main has it; skip if worktree already has `.venv`; fall through to baseline-setup's install otherwise.
  - Symlink command: `ln -s "$(git -C <main> rev-parse --show-toplevel)/.venv" .venv`.
  - Extrapolation line: "for other stacks, do the analogous share-from-main (`node_modules`, `vendor/bundle`, `target`, …); Go and Gradle share caches globally, nothing to link."
  - Follow-up note: if the worktree changes manifest/lockfile, `rm .venv && uv sync` (or the stack's equivalent) to rebuild locally.
  - Respects: IV1, IV2, PC1, AS2, AS3.
- **1.2** `plugins/up/skills/git-worktrees/SKILL.md:49` (modify) — change the pyproject line in `## Baseline setup` to guard on the env not already existing: `[ -f pyproject.toml ] && [ ! -e .venv ] && (uv sync 2>/dev/null || pip install -e .)`. Preserves IV2.
- Commit: `feat(git-worktrees): share main repo env into new worktree`

### Risks / rollback

- RK1 — Reader treats the Python example as the only supported case. Mitigation: the new section leads with the principle; the example is labeled "worked example" and extrapolation is one line below.

## Verify

Result: passed

Positive:
- CK1 — `## Share environment from main` section present in skill (line 44).
- CK2 — Python worked example uses `ln -s "$main/.venv" .venv` with `$main` resolved via `git worktree list --porcelain` (absolute path).
- CK3 — Extrapolation line names `node_modules`, `vendor/bundle`, `target`, and calls out Go / Gradle as globally shared.
- CK4 — Follow-up note instructs `rm .venv && uv sync` when deps diverge.

Invariants / assumptions:
- CK5 (IV1) — Symlink target is absolute; smoke test shows `/Users/.../smoke/.venv` as the link target.
- CK6 (IV2) — `[ ! -e .venv ]` guard present in both the share block (line 52) and baseline pyproject line (line 68); second run of the snippet did not overwrite the existing symlink.

Smoke: ran the documented snippet in a throwaway repo (gitignored `.venv` in main, fresh worktree): `readlink .venv` → absolute path to main's `.venv`; re-running the snippet left the symlink untouched.

## Conclusion

Outcome: `git-worktrees` skill now documents sharing the main repo's environment into a new worktree (Python as worked example, extrapolation to other stacks); 27b43c8, 71cc211.

Invariants:
- IV1 — Symlink example uses absolute path via `$(git worktree list --porcelain | awk '/^worktree / {print $2; exit}')`; smoke test confirmed absolute target.
- IV2 — `[ ! -e .venv ]` guard in both the share block and baseline pyproject line; idempotency smoke confirmed.

### Assumptions check
- AS1 — held (unverified beyond the common case): worktrees of the same repo share the parent's interpreter pin.
- AS2 — held by construction: `git worktree add` from a clean main branches off HEAD, so manifests match at creation.
- AS3 — documented as guidance in the skill; enforcement is on the user.

Review findings:
- Important — baseline pyproject line silenced `uv sync` errors via `2>/dev/null`, violating PC2. Fixed in 71cc211 by dropping the redirect; `|| pip install -e .` alternation retained since it's a try-uv-else-pip path, not a silenced failure.

### Hands-off decisions
- make: size=Medium — hands-off default; skill edit plus doc sweep.
- udesign: symlink-by-default at create-time, local install only if deps diverge later — rationale: user guidance "symlink if deps don't change, else create local env"; at create-time deps are identical by definition.
- udesign: general principle + one worked example, not an ecosystem table — rationale: user guidance "symlink/uv sync is a special case; with different langs do the analogous in your stack".
- uplan: plan auto-approved (hands-off).
- ureview: fixed Important — dropped `2>/dev/null` from baseline pyproject line to honor PC2.

### Deferred (needs user input)
