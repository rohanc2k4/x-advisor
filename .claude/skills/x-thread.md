---
name: x-thread
description: Draft a multi-post thread on a topic. Output is ready to paste; posting via API is constrained (see caveat).
---

# /x-thread

Draft a multi-post thread on a single topic. Same voice machinery as `/x-draft`, scaled up to a numbered chain. The drafter works reliably; the poster has API constraints — read the caveat.

## When to invoke

- Topic too dense for 280 chars.
- You want a sequence: hook → exposition → reveal → CTA.
- After `/x-pulse` or `/x-targets` if the recommendation is "post a thread on X."

## What it does

1. Reads `voice/` and your last 20 posts (same as `/x-draft`).
2. Drafts a numbered thread (default 4-6 posts) on the given topic, each within char limit.
3. Shows the full chain with char counts per post.
4. Asks for approval / edits / redraft / scrap.
5. On approval, saves to `outputs/x-threads-YYYY-MM-DD.md` AND prints a paste-ready block for manual posting (see caveat).

## Inputs

- One argument: the topic.
- Optional `--length N`: target number of posts in the thread (default 5).

## Outputs

- Numbered thread inline in chat.
- Saved to `outputs/x-threads-YYYY-MM-DD.md` on approval.
- A paste-ready, numbered text block under the saved draft so you can copy-paste into X manually (see caveat).

## Procedure

### Phase 1: Voice load

Same as `/x-draft`: read `voice/` + pull last 20 posts via `search_tweets`.

### Phase 2: Outline

Before drafting prose, generate a short outline:
1. Hook (post 1) — what hooks attention
2. Setup (post 2) — what's broken / unobvious / wrong
3. Expand (posts 3-4) — the substance
4. Reveal / CTA (last post) — what to do, where to find more

Show the outline to the user first. Ask if they want to proceed.

### Phase 3: Draft each post

For each outline node:
- Draft the post.
- Each post stands alone enough to read in isolation (because the algorithm may surface single posts out of context).
- Hard cap at 280 chars per post on Free tier; ~25k on Premium.
- No em dashes, no Tier-1 vocab (same rules as `/x-draft`).

### Phase 4: Present full thread

Print as a numbered chain:

```
1/5  <post text>     [chars: N]
2/5  <post text>     [chars: N]
...
```

Below: voice match notes + length notes.

Prompt:

```
[a]pprove / [e]dit post N / [r]edraft all / [s]crap
```

### Phase 5: Save + paste-ready output

On approval:

1. Save to `outputs/x-threads-YYYY-MM-DD.md` with frontmatter:

```yaml
---
title: X Thread YYYY-MM-DD — <topic slug>
type: thread
length: <N posts>
last_updated: YYYY-MM-DD
---
```

2. After the saved thread, append a "paste-ready" block: each post in its own fenced code block, numbered, ready to copy one at a time into X's compose UI.

### Phase 6: Posting (the caveat)

API self-reply restrictions (Feb 2026) mean a true threaded chain via `reply_to_tweet` only works if the original poster mentions / quotes you. Since the original poster is YOU, this technically should be fine in theory, but the restriction has been inconsistent for self-replies on Free / Basic tiers.

Two posting paths:

**Path A — manual (recommended)**: paste each post block from `outputs/x-threads-*.md` into X's compose UI. X's UI treats sequential composes as a thread when you use the "+" add-post button. This is what guilyx and most builders do.

**Path B — programmatic (experimental)**: chain `x-mcp.post_tweet` for post 1, then `x-mcp.reply_to_tweet` for posts 2+, replying to the previous post's ID. May fail on Free / Basic tier per the Feb 2026 restriction. If it fails on post 2, you've posted a standalone tweet without a thread context. Abort and switch to Path A.

If the user explicitly opts into Path B, ship it; otherwise default to Path A.

## Edge cases

- **`--length` too high**: cap at 10. Anything longer is a blog post in costume.
- **Topic too thin for a thread**: suggest `/x-draft` instead.
- **Char overflow on any post**: tighten that post inline, show diff, continue.
