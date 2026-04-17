---
name: data-engineering
description: Use when working on dataset code, data pipelines, or data processing. Enforces sequential stages, idempotent + atomic + retriable units, write-to-temp-then-rename, pre-flight checks, observability. Auto-triggers on dataset directories and pipeline code.
---

# Data Engineering

Follow these principles whenever writing or modifying data pipeline code.

## Pipeline structure — sequential stages, parallelism inside each stage

- Structure code as sequential stages: download → process → encode → build → verify
- Parallelize inside each stage, not across stages
- Prefer `asyncio` for IO-bound operations (API downloads, HTTP requests)
- Prefer `multiprocessing` for compute-bound operations (parsing, encoding)
- Prefer delegating to optimized native tools (ffmpeg, rsync) over reimplementing in Python

## Reliability — idempotent, atomic, retriable

Every stage must be:

- Idempotent: running it twice with the same inputs produces the same output, no side effects
- Atomic: each unit of work (one file, one episode) either fully completes or leaves no partial state. Write to a temp path, then rename to final. Never write directly to the final destination.
- Retriable: a crash mid-stage must be recoverable without restarting from scratch. Persist progress to disk (e.g. `progress.json`), check it on restart, skip completed items.

A crash mid-run should cost minutes of re-work, not hours.

## Never destroy progress without asking

- Never nuke existing data or progress without explicit user consent. Prefer truncating to the last valid state over wiping and rebuilding.
- Use write-to-temp-then-rename instead of overwriting in place
- Cached / intermediate artifacts (e.g. `video_cache/`) are sacred — never delete them
- Only the final build output is expendable and rebuildable from cache

## Pre-flight checks — run before every large operation

<required>
1. Test on 1 item first — always provide a `--limit 1` or equivalent, always run it before the full pipeline
2. Estimate disk usage — calculate expected output size, check available space, fail early if insufficient
3. Set timeouts — every network call, subprocess, and file operation has a timeout. Hanging for 5 hours is worse than crashing in 5 minutes.
4. Sanity-check inputs — validate data shapes, file existence, schema compatibility before processing
</required>

<bad-example>
```
# Launching a 500-episode preprocess without testing
uv run python -m pipeline preprocess --all
```
</bad-example>

<good-example>
```
uv run python -m pipeline preprocess --limit 1
# check output looks correct, then:
uv run python -m pipeline preprocess --all
```
</good-example>

## Observability — log stages, progress bars on long loops

- Logs everywhere — every stage entry/exit, every skip, every error, every retry. Use Python `logging`, not print.
- tqdm for interactive progress — any loop over items that takes more than a few seconds gets a tqdm bar
- Log wall-clock time per stage and per item so bottlenecks are obvious
- Log disk usage before and after large writes

## Concurrency — pick the right primitive for the bottleneck

- API calls, downloads → `asyncio` + `aiohttp` (IO-bound, no GIL concern)
- Video encoding, parsing → `multiprocessing` / subprocess (CPU-bound or native tools)
- ffmpeg, rsync → `subprocess` directly (already optimized — just orchestrate)

<bad-example>
```python
for frame in video:
    resize(frame)
    save(frame)
```
</bad-example>

<good-example>
```python
subprocess.run(["ffmpeg", "-i", input, "-vf", "scale=512:512", output], timeout=300)
```
</good-example>
