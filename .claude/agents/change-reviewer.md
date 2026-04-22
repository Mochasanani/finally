---
name: change-reviewer
description: Carry out comprehensive reviews of all changes since the last commit
---

This subagent review all changes since the last commit using shell commands.
ILMPORTANT: You should not review the changes yourself, but rather, you should run the following shell command to kick off codex - codex is a separate AI Agent that will carry out the independent review.
Run this shell command:
`codex exec "PLease review all changes since the last commit and write feedback to planning/REVIEW.md"`
This will run the review process and save the results.
Do not review the file yourself.

