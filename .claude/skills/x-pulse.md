---
name: x-pulse
description: Daily timeline + analytics digest. Reads the last 24h of your X timeline, mentions, and post metrics and produces a brief.
---

# /x-pulse

Daily kickoff for X. One pass through your timeline, mentions, and analytics; one brief out the other end. Read this on your morning coffee, decide where to spend your replies and posts that day.

## When to invoke

- Once per day, ideally morning.
- After a campaign post lands and you want a quick read on traction.
- Before `/x-debrief` (weekly) to refresh the picture.

## What it does

1. Resolves your `user_id` from `config.yaml` `handle` via x-mcp `get_user` (cached in `.cache/me.json` after first run).
2. Fetches your last 10 posts via `search_tweets` with `from:<handle>` and pulls metrics via `get_metrics`.
3. Fetches mentions via `get_mentions`, then filters locally to the last 24h window.
4. For each niche handle (from `niche.yaml` or `get_following` fallback), fetches recent posts via `search_tweets` and reads `public_metrics` from the response. Does NOT call `get_metrics` on others' posts: that endpoint returns non-public fields only for posts you authored.
5. Synthesizes a one-page brief and writes it to `outputs/x-pulse-YYYY-MM-DD.md`.

## Inputs

- None. Reads `config.yaml` for `handle` and `timezone`; reads live from x-mcp.

## Outputs

- `outputs/x-pulse-YYYY-MM-DD.md`, the brief.
- A short summary printed inline with today's top recommendation.

## Procedure

### Phase 0: Resolve identity

Load `config.yaml`. Required: `handle`, `timezone`. If `user_id` is set, use it. Otherwise:

```
x-mcp.get_user({ username: <handle> })
```

Cache `user_id` in `.cache/me.json` so subsequent runs skip this step.

If `config.yaml` is missing, print one-line guidance: copy `config.example.yaml` to `config.yaml`, fill in your handle.

### Phase 1: Fetch your own recent posts

```
x-mcp.search_tweets({
  query: "from:<handle> -is:retweet",
  max_results: 10,
})
```

For each returned tweet ID, fetch metrics:

```
x-mcp.get_metrics({ tweet_id: <id> })
```

These return both `public_metrics` (likes, retweets, replies, quotes; bookmark_count on Basic+ tier) and `non_public_metrics` / `organic_metrics` (impressions, user_profile_clicks, url_link_clicks) when you authored the post.

Stop calling `get_metrics` if a 429 fires. Note the reset timestamp.

### Phase 2: Fetch mentions

```
x-mcp.get_mentions({ max_results: 100 })
```

Loop with `next_token` until you have one full page past the 24h cutoff, then stop. Filter locally by `created_at >= now - 24h`.

If the loop returns more than 5 pages without crossing the cutoff, bail and mark the section `truncated: true` with the last `next_token` saved to frontmatter.

### Phase 3: Fetch in-niche signal

For each handle in `niche.yaml` (or the top 20 of your `get_following` if `niche.yaml` absent), call:

```
x-mcp.search_tweets({
  query: "from:<niche_handle> -is:retweet -is:reply",
  max_results: 5,
})
```

Run these in batches of 10 in parallel. Filter results locally to posts younger than 24h. Pull `public_metrics` from the search response. Do NOT call `get_metrics` on others' posts.

### Phase 4: Synthesize

For each section:
- **Top of feed (5)**: niche posts in the last 24h sorted by engagement velocity, computed as `(public_metrics.like_count + public_metrics.reply_count + public_metrics.retweet_count) / max(age_minutes, 1)`. Pick 5.
- **Mentions**: every account that @mentioned you in the window. Mark each `reply` / `like` / `skip` based on whether engagement is warranted.
- **Your post performance**: your last 10 posts with nested fields `public_metrics.like_count`, `public_metrics.reply_count`, `public_metrics.retweet_count`, `public_metrics.quote_count`, `non_public_metrics.impression_count` (when present), `non_public_metrics.user_profile_clicks` (when present). Compare each metric against your 30-day median for that metric (fall back to "first run" if no prior pulses exist). Mark `above` / `at` / `below`.

### Phase 5: Recommend

Generate 1-3 concrete actions. Each a single sentence with a verb and a target.

Examples:
- "Reply to @user_x's post from 12 min ago (5k engagement, 80k follower author): open thread, your take on RAG fits."
- "Draft a thread on supersede: 3 niche posts in the last hour are discussing wiki-rot."

### Phase 6: Write the brief

Write `outputs/x-pulse-YYYY-MM-DD.md` with this frontmatter:

```yaml
---
title: X Pulse YYYY-MM-DD
type: pulse
since: <ISO of window start, in local timezone>
until: <ISO of window end>
handle: <@handle>
user_id: <numeric id>
timezone: <IANA>
truncated: false
last_updated: YYYY-MM-DD
---
```

If any section was truncated, set `truncated: true` and add a `next_tokens:` map with the token per section.

Then four sections matching the synthesis: Top of feed, Mentions, Your post performance, Today's recommendation.

### Phase 7: Echo

Print the recommendation section inline with the full-file path.

## Edge cases

- **First run, no `config.yaml`**: stop with one-line guidance.
- **`get_user` returns 404 for the handle**: stop; the handle is wrong.
- **Rate limited mid-fetch**: write what you have; mark missing sections with `[rate-limited until <ISO>]`.
- **Empty timeline + empty mentions + zero recent own-posts**: bail with a one-line brief explaining no signal.
- **No prior pulses for baseline comparison**: write `baseline: first-run` in the post-performance section and skip the above/at/below tagging.
