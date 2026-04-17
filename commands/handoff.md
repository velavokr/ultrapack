Produce a handoff summary so another agent session can continue this work with zero context beyond CLAUDE.md and the codebase.

**If an active task file exists at `docs/tasks/<slug>.md`:** append the handoff to its `## Conclusion` under a `### Handoff — YYYY-MM-DD` subsection. Otherwise, produce a standalone summary (print it and, if the user wants, save to `tmp/handoff.md`).

Before writing, gather concrete state:
- Read `git status` and `git diff` for uncommitted work
- Check for active remote sessions (tmux, running processes) if applicable
- Review the conversation for decisions made, approaches tried, and blockers hit
- Check for temp files or logs in `tmp/` that contain useful state

Write the handoff with these sections (omit a section only if genuinely not applicable):

1. **Goal** — one or two sentences: the top-level objective, not the latest sub-task
2. **Problem** — what's broken or missing, why it matters, concrete example if available. The next agent has no conversation context — explain the problem well enough that they could diagnose it independently.
3. **Infrastructure / Environment** — SSH commands, pod IPs, tmux sessions, required env vars and where they live, exports/source commands, tools/binaries needed. Omit for purely local work.
4. **Current state** — what's been done: files modified (with paths), pipeline stages completed, data/artifacts produced, what is working and verified
5. **Active blocker** — the specific problem preventing progress: what it is, what was tried, partial fixes applied (with file:line), diagnostic output. If none, say "no blockers"
6. **Key files** — only files the next session will need to touch or reference
7. **What to do next** — numbered, actionable steps in order. Include commands where possible. Most important section after Goal.
8. **Gotchas** — non-obvious things that waste time: env quirks, sync commands, versioning traps, flaky behavior

Rules:
- Be concrete: exact commands, exact paths, exact error messages. No "run the appropriate command."
- Be terse: bullets over prose. No filler.
- Only include info that can't be derived from code or git history. Don't restate CLAUDE.md.
- Include copy-pasteable commands over descriptions.
- If unsure about something (e.g. whether a remote process is still running), say so explicitly rather than assuming.
