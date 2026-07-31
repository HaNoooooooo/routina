---
name: hello-test
description: Minimal test skill used to verify that a Claude Code cloud routine (created via /schedule) can load and execute a custom skill from its checked-out git repository. Use only when explicitly asked to run the hello-test skill.
---

# hello-test

When invoked, do exactly this and nothing else:

1. Get the current UTC time.
2. Output a single line in this exact format:
   `SKILL_TEST_OK: hello-test skill executed at <UTC timestamp>`
3. Stop. Do not perform any other actions, file edits, or tool calls beyond what is needed to get the timestamp.
