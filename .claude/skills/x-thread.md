---
name: x-thread
description: Draft a multi-post thread on a topic. Posting is manual paste only (API self-reply restrictions make programmatic threading unreliable).
---

# /x-thread

Draft a multi-post thread on a single topic. Same voice machinery as `/x-draft`, scaled up to a numbered chain. Posting is manual paste; programmatic posting is not supported on self-serve API tiers.

## When to invoke

- Topic too dense for 280 chars.
- You want a sequence: hook, exposition, reveal, CTA.
- After `/x-pulse` or `/x-targets` if the recommendation is "post a thread on X."

## What it does

1. Reads `voice/` (excluding any `README.md`) and your last 20 posts.
2. Drafts a numbered thread (default 5 posts) on the given topic, each within the char limit.
3. Shows the full chain inline with per-post char counts.
4. Prompts `[a]pprove` / `[e]dit post N` / `[r]edraft all` / `[s]crap`.
5. On approval, saves to `outputs/x-threads-YYYY-MM-DD.md` AND prints a paste-ready block: each post in its own fenced block, ready to paste one at a time into X's compose UI.

## Inputs

- One argument: the topic.
- Optional `--length N`: target number of posts (default 5, capped at 10).

## Outputs

- Numbered thread inline.
- Saved to `outputs/x-threads-YYYY-MM-DD.md` on approval.
- Paste-ready block printed under the saved file path so you can copy each post sequentially.

## Why manual paste only

The X API restricts programmatic self-replies on all self-serve tiers as of Feb 2026. A "thread" via API is technically a chain of self-replies; on Free/Basic/Pro tiers the second post will fail or detach from the thread. Manual paste into the X compose UI works reliably: the `+` button to add a post stitches them into a proper thread.

We deliberately do not expose a programmatic posting path for `/x-thread`. It's a drafter. You paste.

## Procedure

### Phase 1: Voice load

Same as `/x-draft`: read voice files (excluding any `README.md` at any depth under `voice/`) and last 20 posts via `search_tweets`.

### Phase 2: Outline

Generate a short outline before drafting prose:
1. Hook (post 1): what catches attention.
2. Setup (post 2): what's broken or unobvious.
3. Expand (posts 3-4): the substance.
4. Reveal / CTA (last post): what to do, where to find more.

Show the outline. Ask if the user wants to proceed.

### Phase 3: Draft each post

For each outline node:
- Draft the post.
- Each post stands alone enough to read in isolation (algorithm may surface single posts out of context).
- Hard cap at 280 chars per post on Free tier; 25,000 on Premium per `config.yaml`.
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

On approval, save to `outputs/x-threads-YYYY-MM-DD.md` with frontmatter:

```yaml
---
title: X Thread YYYY-MM-DD, <topic slug>
type: thread
handle: <@handle>
length: <N>
last_updated: YYYY-MM-DD
---
```

Then the numbered thread, then a paste-ready section: each post in its own fenced code block (use `~~~` fence), numbered, with explicit "post 1 of N" / "post 2 of N" labels so you can paste sequentially without losing track.

## Edge cases

- **`--length` over 10**: cap at 10. Anything longer is a blog post in costume; redirect to a Hashnode / dev.to write-up.
- **Topic too thin**: suggest `/x-draft` instead.
- **Char overflow on any post**: tighten that post inline, show diff, continue.
- **Voice corpus empty**: stop and prompt to seed `voice/`.
