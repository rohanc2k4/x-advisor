---
name: x-post
description: Post a drafted message via the x-mcp post_tweet tool. Typed hash-code confirmation required.
---

# /x-post

Ship a drafted post. Confirmation gate is mandatory and uses a per-post code phrase to defeat replay and visual-glance approvals.

## When to invoke

- Directly from `/x-draft` after approval.
- On its own with `latest` to ship the most recent unposted approved draft.
- On its own with explicit text to ship something you wrote outside the agent.

## What it does

1. Resolves the draft text (passed verbatim, or `latest` -> most recent unposted `## Approved` entry in today's drafts log).
2. Renders the draft inside a fenced code block. Default fence delimiter is `~~~`. If the draft contains `~~~` AND triple-backticks, escape inner `~~~` with a zero-width space prefix.
3. Computes a 4-character confirmation code from the SHA-256 of the final text (first 4 hex chars, lowercase).
4. Requires the user to type that exact 4-character code (case-insensitive) to ship. Any other input cancels.
5. On confirmation, calls x-mcp `post_tweet`.
6. Updates the drafts log: marks the entry `posted: true` with `post_id` + `post_url`.
7. Logs to `outputs/x-posted-YYYY-MM-DD.md`.

## Inputs

- One argument: the draft text in quotes, or `latest`.
  - `/x-post "Folks: ..."`
  - `/x-post latest`

## Outputs

- On success: post URL printed inline; entries updated in `outputs/x-drafts-YYYY-MM-DD.md` and `outputs/x-posted-YYYY-MM-DD.md`.
- On rejection: nothing happens. Drafts stay unposted.

## Procedure

### Phase 1: Resolve draft

If argument is `latest`: read `outputs/x-drafts-YYYY-MM-DD.md` (today's date in local timezone). Find the most recent `## Approved` entry with `posted: false`. If none, abort with "no unposted drafts in today's log."

If argument is a quoted string: use it directly. Skip the drafts-log update step later.

### Phase 2: Present + verify

Compute `code = sha256(draft_text).hex[:4]` (lowercase hex). The input is the raw draft bytes UTF-8 encoded.

Choose a fence delimiter that doesn't appear in the draft. Default is `~~~`. If the draft contains `~~~`, switch to triple-backticks. If it contains both, prefix each offending fence inside the draft with a zero-width space (`​`) so the rendered block isn't terminable by content from the draft.

Render:

```
draft preview:
~~~
<draft text verbatim>
~~~
chars: <N> / <limit>
tier: <Free | Premium>
hash code: <abcd>

Type the 4-character hash code to ship. Anything else cancels.
```

### Phase 3: Branch

- Input matches `code` exactly (case-insensitive comparison, leading/trailing whitespace trimmed): proceed to Phase 4.
- Input is the wrong code, or contains anything else (including `post`, `yes`, `y`, an empty line): print "cancelled, nothing posted. Hash code did not match." and exit.

Per-post hash codes defeat:
- Replay attacks (a copy-pasted `post` from a previous turn).
- Glance approvals (the user must look at THIS draft's hash, which is unique to its content).
- Adversarial drafts containing a fake confirmation prompt (the model can't predict the hash).

### Phase 4: Post

Call:

```
x-mcp.post_tweet({ text: <draft> })
```

If 403, surface the error verbatim and abort. Most common cause: token doesn't have write scope (see x-mcp README troubleshooting).

If 429, surface the reset timestamp and abort.

If the response includes a duplicate-content error ("Status is a duplicate"), surface it, suggest editing.

### Phase 5: Update drafts log (if `latest` path)

In `outputs/x-drafts-YYYY-MM-DD.md`, find the entry just posted. Update:
- `posted: false` to `posted: true`
- Add `post_id: <id>` and `post_url: <url>` immediately below `posted:`.

Leave the draft text and timestamp untouched.

### Phase 6: Log to posted artifact

Append to `outputs/x-posted-YYYY-MM-DD.md`. Frontmatter only on first write of the day:

```yaml
---
title: X Posts YYYY-MM-DD
type: posted
handle: <@handle>
last_updated: YYYY-MM-DD
---
```

Entry:

```
## HH:MM:SS
url: <post_url>
chars: <N>
text:
~~~
<draft text verbatim>
~~~
```

### Phase 7: Echo

Print the post URL inline as a clickable link.

## Edge cases

- **Draft over char limit**: refuse to post; suggest trimming or `/x-thread`.
- **Network error mid-post**: surface, don't retry; the user reinvokes manually.
- **Drafts log missing for today**: if `latest`, abort with the missing-file message; if direct text, no log update step runs (skip Phase 5).
- **Hash code collides with an English word**: rare with 4 hex chars (16^4 = 65k); acceptable. The user types the code, not a meaningful confirmation phrase.

## Why a per-post hash

A typed phrase like `post` is too easy to mis-key into. A static phrase is also subject to replay if the model emits a fake confirmation prompt inside the draft preview. A hash of the exact final text means:
- The code is unique per draft (no replay across posts).
- The user must look at the displayed hash (no muscle memory).
- An adversarial draft can't predict the hash, so it can't pre-bake a fake confirmation.
