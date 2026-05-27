---
name: x-targets
description: Find high-value reply targets in your niche. Surfaces fresh, fast-moving posts where a manual reply lands well.
---

# /x-targets

Surface fresh, fast-moving posts from accounts in your niche where a thoughtful manual reply puts your voice in front of a real audience. Premium reply-boost amplifies the value of this list.

## When to invoke

- After `/x-pulse` if you want to spend the next hour replying.
- Before opening X, so you arrive with a target list instead of scrolling into one.
- 2-3 times per day during active growth; once per day at steady state.

## What it does

1. Resolves your `user_id` from `config.yaml` (same flow as `/x-pulse`).
2. Reads `niche.yaml` if present; otherwise pulls `get_following({ user_id })` and caps at 100.
3. For each niche handle, calls `search_tweets` filtered to that author's recent posts.
4. Filters results locally to posts younger than the configured max age (default 2h).
5. Ranks by engagement velocity computed from `public_metrics` returned in the search response. Does NOT call `get_metrics` on others' posts: that endpoint returns non-public fields only for posts you authored.
6. Returns the top N with a one-sentence read on why each is worth a reply.

## Inputs

- Optional first arg: max age in hours (default 2).
- Optional second arg: max results (default 10).

## Outputs

- Ranked list printed inline, each entry with:
  - Post URL
  - Author handle and niche tag
  - Post content (first 200 chars)
  - Engagement velocity
  - One-line "why this is worth replying to" note
- Saved to `outputs/x-targets-YYYY-MM-DD-HHMM.md`.

## Procedure

### Phase 0: Resolve identity

Same as `/x-pulse` Phase 0. Reads `handle` and resolves `user_id` (cached in `.cache/me.json`).

### Phase 1: Resolve the niche set

Check for `niche.yaml` at repo root. Format:

```yaml
niche:
  - handle: simonw
    tag: ai-tooling
  - handle: jasonzhou
    tag: ai-agents
```

If absent, fall back to:

```
x-mcp.get_following({ user_id: <me>, max_results: 100 })
```

Paginate with `next_token` once if results are sparse.

### Phase 2: Fetch fresh posts per niche handle

For each handle, in batches of 10 parallel:

```
x-mcp.search_tweets({
  query: "from:<handle> -is:retweet -is:reply",
  max_results: 5,
})
```

The response includes `public_metrics` per tweet AND a `created_at` field. Filter locally to `created_at >= now - max_age_hours`.

If any call returns 429, abort the batch, record the cutoff, return whatever you collected.

### Phase 3: Rank by velocity

For each post still in the window:

```
engagement_velocity =
  (public_metrics.like_count
   + public_metrics.reply_count
   + public_metrics.retweet_count
   + public_metrics.quote_count) / max(age_minutes, 1)
```

Sort descending. Take top N (default 10).

### Phase 4: Explain each

For each ranked post, generate a one-sentence reason it's worth replying to. Be concrete:

- "Polarizing take, reply thread is shallow, room for a substantive contrarian reply."
- "Author has 80k followers, post is 15 min old, only 3 replies. High visibility, low competition."
- "Direct question post; a specific worked-example answer reads as expert."

### Phase 5: Print + save

Print the ranked list inline. Save to `outputs/x-targets-YYYY-MM-DD-HHMM.md` with frontmatter:

```yaml
---
title: X Targets YYYY-MM-DD HH:MM
type: targets
generated_at: <ISO local>
max_age_hours: <N>
truncated: false
last_updated: YYYY-MM-DD
---
```

Set `truncated: true` if you abandoned any per-handle fetch.

## Edge cases

- **`get_following` empty**: prompt the user to seed `niche.yaml` or follow some accounts first.
- **No fresh posts**: bump `max_age_hours` once and retry. If still empty, return "your niche is quiet right now."
- **Rate limited mid-batch**: return what you have, mark `truncated: true`.
- **`public_metrics` absent on a search result**: rare, but if it happens skip that post. Do NOT call `get_metrics` on others' tweets to backfill (it won't return non-public fields).

## Why this works

X's algorithm boosts the original poster's most-engaged replies into adjacent timelines. A thoughtful early reply on a high-velocity post rides that post's reach for free. Premium amplifies your reply's position. The MCP can't reply for you (Feb 2026 API restriction), but it puts you in front of the right targets at the right time.
