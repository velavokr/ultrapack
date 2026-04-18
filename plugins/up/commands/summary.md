---
description: Produce a summary so another session can continue with zero context beyond CLAUDE.md, the codebase, and the summary itself. Asks whether to append to the current task file or create a new summary task file.
---

# /up:summary

Prepare a handoff summary so another agent session can continue this work without the current conversation. The drafting is delegated to the `up:summarizer` subagent (pinned to Sonnet); the main session owns only the destination choice and the file write.

## Process

### 1. Dispatch the `up:summarizer` subagent

<required>
Drafting runs in the subagent, not in the main session. Dispatching is not optional — drafting in the main session defeats the performance motivation (main sessions typically run on Opus; the subagent runs on Sonnet).
</required>

Detect the active task file first:

```bash
ls -t docs/tasks/*.md 2>/dev/null | head
```

The active task file is the most-recently-modified entry whose `**Status:**` header is not `done`. If none qualify, pass `null`.

Dispatch via the Task tool with:
- `subagent_type: up:summarizer`
- Working directory (absolute).
- Active task file path, or `null`.

Do not pass the conversation history. The subagent rebuilds state from the repo by design.

### 2. Receive the draft

The subagent returns prose beginning with the line `Draft summary below — main session decides destination.` followed by the eight-section summary (Goal / Problem / Infrastructure / Current state / Active blocker / Key files / What to do next / Gotchas).

Quote the draft verbatim back to the user. Do not rewrite it — if something is missing, ask the subagent to revise rather than silently patching on Opus.

### 3. Ask the user where to put it

<required>
After showing the draft, ask:

1. Append to the current task file's `## Conclusion` as a `### Summary — YYYY-MM-DD` subsection (provide the detected `docs/tasks/<slug>.md` path).
2. Create a new file at `docs/tasks/summary-<new-slug>.md` (propose a slug based on the current work).

Pick the destination based on the user's answer. Do not write anywhere without confirmation.
</required>

If no active task file was detected in step 1, option 1 is unavailable — only offer option 2.

### 4. Write

Perform the write in the main session using Edit (append) or Write (new file). The subagent has no write tools.

## Rules

- Subagent drafts; main session asks and writes. Never let drafting slip back into the main session — that reintroduces the Opus cost this command exists to avoid.
- Concrete: exact commands, exact paths, exact error messages. No "run the appropriate command."
- Terse: bullets over prose. No filler.
- Include only info that can't be derived from code or git history. Don't restate `CLAUDE.md`.
- Never write without confirmation.
