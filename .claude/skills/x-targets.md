---
name: x-targets
description: Find high-leverage reply targets. Searches niche accounts for fresh posts where your manual reply lands well.
---

# /x-targets

Surface fresh, high-velocity posts from accounts in your niche where a thoughtful manual reply puts your voice in front of a real audience. Premium reply-boost amplifies the value of this list.

## When to invoke

- After `/x-pulse` if you want to spend the next hour replying.
- Before opening X, so you arrive with a target list instead of doom-scrolling into one.
- 2-3 times per day for active growth phase; once per day for steady-state.

## What it does

1. Pulls your `get_following` list via x-mcp.
2. Cross-references against `niche.yaml` (a small registry of accounts you consider in-niche; defaults to all of `get_following` if absent).
3. For each in-niche account, calls `search_tweets` with `from:<handle> -is:retweet -is:reply` filtered to posts younger than 2 hours.
4. Pulls engagement metrics on each (via `get_metrics`).
5. Ranks by engagement velocity = `(likes + replies + retweets) / age_in_minutes`.
6. Returns the top 10 with a one-sentence read on why each is worth a reply.

## Inputs

- Optional first arg: max age in hours (default 2).
- Optional second arg: max results (default 10).

## Outputs

- A ranked list printed in the chat, each entry with:
  - Post URL
  - Author handle, follower count, niche tag
  - Post content (first 200 chars)
  - Engagement velocity
  - One-line "why this is worth replying to" note
- Saved to `outputs/x-targets-YYYY-MM-DD-HH.md`.

## Procedure

### Phase 1: Resolve the niche set

Check for `niche.yaml` at repo root. Format:

```yaml
niche:
  - handle: simonw
    tag: ai-tooling
  - handle: jasonzhou
    tag: ai-agents
  - handle: theo
    tag: ai-tooling
```

If absent, fall back to the user's `get_following` list (cap at 100).

### Phase 2: Fetch fresh posts

For each handle in the niche set, in batches of 10 parallel requests:
```
x-mcp.search_tweets({
  query: "from:<handle> -is:retweet -is:reply",
  max_results: 5
})
```

Filter results to posts created within the `max_age_hours` window (default 2h).

### Phase 3: Score by velocity

For each post, get its metrics:
```
x-mcp.get_metrics({ tweet_id: <id> })
```

Compute `engagement_velocity = (likes + replies + retweets) / max(age_minutes, 1)`.

### Phase 4: Rank and explain

Sort descending by velocity. Take the top N (default 10). For each, generate a one-sentence reason it's worth replying to. Reasons should be concrete:

- "It's a polarizing take and the reply thread is shallow, so a substantive contrarian reply has room to land."
- "Author has 80k followers, the post is 15 minutes old, only 3 replies so far. High visibility, low competition."
- "Question post — direct answer with an example from your work will read as expert."

### Phase 5: Print and save

Print the ranked list in the chat. Save the same list to `outputs/x-targets-YYYY-MM-DD-HH.md` with frontmatter:

```yaml
---
title: X Targets YYYY-MM-DD HH:MM
type: targets
generated_at: <ISO>
max_age_hours: <N>
last_updated: YYYY-MM-DD
---
```

## Edge cases

- **`get_following` empty**: ask the user to seed `niche.yaml` or follow some accounts.
- **No fresh posts**: increase `max_age_hours` once and retry. If still empty, return "your niche is quiet right now, check back in an hour."
- **Rate limited mid-fetch**: return what you have, note the cutoff.

## Why this works

Twitter's algorithm boosts the original poster's most-engaged replies into adjacent timelines. A thoughtful early reply on a post with high velocity gets carried by that post's reach for free. Premium amplifies your reply's position in the thread. The MCP can't reply for you (API restriction), but it puts you in front of the right targets at the right time.
