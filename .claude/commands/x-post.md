---
description: Post a drafted message via x-mcp. Per-post hash-code confirmation required.
argument-hint: <draft-text> | latest
---

Invoke the [[x-post]] skill with the user's draft as the argument: `$ARGUMENTS`. If `latest`, reads from today's `outputs/x-drafts-YYYY-MM-DD.md`. Confirmation: agent computes a 4-character hex hash of the final text and the user must type it to ship.
