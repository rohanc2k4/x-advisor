---
name: x-debrief
description: Weekly engagement review. What worked, what didn't, what to change next week.
---

# /x-debrief

Sunday review. Pulls the last 7 days of your posts and their metrics, compares against your own past weeks, identifies what landed and what didn't, recommends specific changes for next week.

## When to invoke

- Sunday evening or Monday morning, weekly.
- After a campaign or launch post to assess.
- Before adjusting `voice/`.

## What it does

1. Resolves your `user_id` and `timezone` from `config.yaml`.
2. Computes the ISO week boundary in your configured timezone.
3. Pulls your recent posts via `search_tweets`, filters locally to the week window.
4. Fetches metrics per post via `get_metrics`.
5. Reads prior `outputs/x-debrief-YYYY-Www.md` files for week-over-week comparison.
6. Categorizes posts by performance against your personal 30-day median.
7. Pattern-matches what wins and misses share.
8. Generates 2-4 concrete next-week actions.
9. Writes to `outputs/x-debrief-YYYY-Www.md`.

## Inputs

- Optional: `YYYY-Www` to debrief any historical week within the recent-search window. Defaults to current ISO week.

## Outputs

- `outputs/x-debrief-YYYY-Www.md`, the debrief.
- Headline summary printed inline: top 3 wins, top 3 misses, next-week actions.

## API constraint to know

`search_tweets` in x-mcp accepts only `query`, `max_results`, and `next_token`. It does NOT accept `start_time` / `end_time`. The X recent-search endpoint covers the last 7 days. We fetch as much as needed and filter by `created_at` locally.

For weeks older than ~7 days, the recent-search endpoint won't return those posts. The skill falls back to reading prior `outputs/x-pulse-*.md` and `outputs/x-posted-*.md` files for the requested historical window. If neither exists for an old week, the debrief surfaces "no data" for that range honestly.

## Procedure

### Phase 0: Resolve identity + timezone

Load `config.yaml`. Required: `handle`, `timezone`. Resolve `user_id` (cached in `.cache/me.json`).

### Phase 1: Compute the window

In the configured timezone, compute Monday 00:00 to Sunday 23:59 for the target ISO week. Store both ends as ISO strings with offset. Compute `YYYY-Www` from ISO week-year (NOT calendar year). January 1 can belong to week 52 of the prior ISO year; late December can belong to week 1 of the next ISO year. Use Python's `isocalendar()` semantics or equivalent.

### Phase 2: Pull posts in window

```
x-mcp.search_tweets({
  query: "from:<handle>",
  max_results: 100,
})
```

Paginate with `next_token` until you have posts older than the window start or you cross 3 pages. Filter locally to `window_start <= created_at <= window_end`.

If pagination exits without crossing the window (all results are inside the window AND there might be more), mark `truncated: true` in frontmatter.

If the requested week is outside the recent-search range, skip this phase and read from local artifacts (`outputs/x-posted-*.md` files dated inside the window).

### Phase 3: Fetch metrics per post

For each post in the window:

```
x-mcp.get_metrics({ tweet_id: <id> })
```

Returns `public_metrics`, `non_public_metrics` (impressions, user_profile_clicks, url_link_clicks), and `organic_metrics` for posts you authored.

Field names to use (nested, not flat):
- `public_metrics.like_count`
- `public_metrics.reply_count`
- `public_metrics.retweet_count`
- `public_metrics.quote_count`
- `public_metrics.bookmark_count` (Basic+ tier only)
- `non_public_metrics.impression_count`
- `non_public_metrics.user_profile_clicks`
- `non_public_metrics.url_link_clicks`
- `organic_metrics.impression_count` (organic-only impression breakdown)

Stop and mark `truncated_metrics: true` if 429 fires.

### Phase 4: Compute baselines

Pull the previous 4 weeks of posts via the same search (further pages of pagination). Compute medians for each metric. These are your personal baselines.

If less than 4 weeks of data exist (early tenure), mark baselines `early_tenure: true` and use whatever is available.

### Phase 5: Categorize

For each post in the debrief window, flag each metric as `above` / `at` / `below` baseline.

Compute an overall score:

```
score = 1 * impression_flag
      + 3 * reply_flag
      + 2 * profile_clicks_flag
      + 2 * retweet_flag
      + 1 * like_flag
```

Where each flag is `+1` if above baseline, `0` if at, `-1` if below.

Rank posts by score. Top 3 = wins. Bottom 3 = misses.

### Phase 6: Pattern-match

Across the wins, identify shared traits:
- Topic cluster (agentic-brain, sanji, build-in-public, replies)
- Format (single post, thread, quote-tweet)
- Hook style (opener pattern)
- Time of day posted (use the configured timezone, NOT UTC)
- Day of week

Same across misses.

### Phase 7: Recommend

Generate 2-4 concrete next-week actions. Each one specific enough to execute without re-deciding:

- "Threads about supersede / promote-memory outperformed single posts on the same topics 3x. Default to threads for technical depth next week."
- "Posts at 7-9am ET got 2.4x the impressions of posts at noon ET. Shift the weekly schedule."

### Phase 8: Write the debrief

`outputs/x-debrief-YYYY-Www.md`:

```yaml
---
title: X Debrief YYYY-Www
type: debrief
handle: <@handle>
timezone: <IANA>
week_start: <ISO with offset>
week_end: <ISO with offset>
posts_in_window: <N>
early_tenure: false
truncated: false
truncated_metrics: false
last_updated: YYYY-MM-DD
---
```

Sections:
1. Headline (3 sentences): posts shipped, impressions delta vs last week, profile-clicks delta.
2. Wins (top 3 with metrics + why).
3. Misses (bottom 3 with metrics + why).
4. Patterns (what united the wins; what united the misses).
5. Next week (the 2-4 recommended actions).

### Phase 9: Echo

Print the headline and the next-week actions inline. Link to the full file.

## Edge cases

- **No posts in window**: write a one-line debrief: "no posts this week, no signal to debrief."
- **Less than 4 prior weeks**: `early_tenure: true`, baselines are noisier; flag in the debrief body.
- **Bookmark unavailable (Free tier)**: skip that dimension, note in body.
- **Outlier post (10x median impressions)**: surface explicitly. Sometimes the lesson is "do that again"; sometimes "lucky algorithm hit, don't over-fit." Mark it; don't let it dominate baseline math.
- **Historical week outside recent-search range**: fall back to local artifacts; mark `data_source: artifacts` in frontmatter.
- **ISO year boundary**: late December / early January weeks can belong to a different ISO year than calendar year. Always compute `YYYY-Www` from ISO week-year, not calendar year.

## Why the personal baseline

X is a popularity machine. Comparing to bigger accounts is discouraging and noisy. Comparing to past-you measures whether you're actually improving the craft. The latter is the only signal you can act on.
