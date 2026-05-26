---
name: x-post
description: Post a drafted message via the x-mcp post_tweet tool. Diff-and-approve gate first.
---

# /x-post

Ship a drafted post. Confirmation gate is mandatory; this is the only skill that writes to the world.

## When to invoke

- Directly from `/x-draft` after approval.
- On its own if you've already drafted manually and want to use the MCP to post.

## What it does

1. Takes a draft string (passed as argument, piped from `/x-draft`, or pulled from `outputs/x-drafts-YYYY-MM-DD.md`).
2. Shows it inside a fenced plain-text block with char count and (if Premium) a long-post indicator.
3. Asks for a typed confirmation. NOT a single-key prompt. Requires literal `post` to ship.
4. On confirmation, calls x-mcp `post_tweet`.
5. Logs the result (post ID, URL, timestamp) to `outputs/x-posted-YYYY-MM-DD.md`.

## Inputs

- One argument: the draft text. Examples:
  - `/x-post "Folks: my second brain runs on claude code. ..."`
  - `/x-post latest` → reads the last drafted post from `outputs/x-drafts-YYYY-MM-DD.md`.

## Outputs

- On success: a one-line confirmation with the post URL.
- On rejection: nothing happens. Drafts stay in their file.

## Procedure

### Phase 1: Resolve draft

If the argument is `latest`, read the most recent draft from today's `outputs/x-drafts-YYYY-MM-DD.md` under the `## Approved` section. If no draft is found, abort.

If the argument is a quoted string, use it directly.

### Phase 2: Present + verify

Show the draft in a fenced plain-text block. Underneath:

```
Char count: <N>
Tier: Free (280 char limit) | Premium (25,000 char limit)
```

Then prompt:

```
Type 'post' to ship, anything else cancels.
```

### Phase 3: Branch

- Input is literally `post` → proceed to Phase 4.
- Anything else (including `yes`, `y`, `confirm`, empty) → print "cancelled, nothing posted" and exit.

This is intentional. The friction is the point. Premium accounts can post in bulk; you should not.

### Phase 4: Post

Call:
```
x-mcp.post_tweet({ text: <draft> })
```

If the response is a 403, surface the error and abort. Most common cause: token doesn't have write scope (see x-mcp README troubleshooting).

If 429 rate-limited, surface the reset timestamp and abort.

### Phase 5: Log

Append to `outputs/x-posted-YYYY-MM-DD.md` with frontmatter only on first write of the day:

```yaml
---
title: X Posts YYYY-MM-DD
type: posted
last_updated: YYYY-MM-DD
---
```

Each post gets its own entry:

```
## HH:MM:SS
URL: https://x.com/<user>/status/<id>
Char count: <N>
Text:
> <draft>
```

### Phase 6: Echo

Print the post URL in chat as a clickable link.

## Edge cases

- **Draft over char limit**: refuse to post, prompt to trim or use `/x-thread`.
- **Network error mid-post**: surface, don't retry. Manual re-invoke is fine.
- **Duplicate post**: x-mcp typically returns 403 "Status is a duplicate." Surface verbatim, suggest editing.

## Why typed confirmation

A single-keypress `y` is too easy. You'll mis-key your way into posting drafts you didn't mean to. Typing `post` takes 4 keystrokes and is impossible to do accidentally.
