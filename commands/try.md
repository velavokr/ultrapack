Manually test the latest code change. Goal is speed — shortest path to confidence.

Steps:
1. Identify the change: check `git diff` or recent conversation context. Focus on behavior, not lines.
2. Design exactly two test cases:
   - **Positive**: happy path, realistic minimal input
   - **Negative**: should fail/be rejected — most likely misuse or edge case
3. Set up environment if needed (start server, build, etc.)
4. Run both tests. Capture output as evidence — don't just say "it worked."
5. If either fails unexpectedly, stop and report before attempting fixes.

Report format:
```
Positive: [PASS/FAIL] — <what happened>
Negative: [PASS/FAIL] — <what happened>
```

Rules:
- This is manual testing. Don't write unit tests.
- Prefer the fastest path: `curl` over Postman, direct CLI over scripted harness.
- If both pass, the change is verified. If either fails, ask how to proceed.
