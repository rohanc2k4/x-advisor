---
name: x-debrief
description: Weekly engagement review. What worked, what didn't, what to change next week.
---

# /x-debrief

Sunday review. Pulls last 7 days of post metrics, compares against prior weeks, identifies what landed and what didn't, recommends changes for next week.

## When to invoke

- Sunday evening or Monday morning, weekly.
- After a campaign / launch post to assess.
- Before adjusting voice training in `voice/`.

## What it does

1. Pulls metrics for all your posts in the last 7 days via x-mcp.
2. Reads prior `outputs/x-debrief-YYYY-Www.md` files for comparison.
3. Categorizes posts by performance (above / at / below your 30-day median).
4. Identifies patterns in what landed (topic, format, time of day, hook style).
5. Identifies patterns in what flopped (same dimensions).
6. Generates 2-4 concrete actionable changes for next week.
7. Writes to `outputs/x-debrief-YYYY-Www.md`.

## Inputs

- Optional: `YYYY-Www` to debrief any historical week. Defaults to current ISO week.

## Outputs

- `outputs/x-debrief-YYYY-Www.md` — the debrief.
- Headline summary printed inline: top 3 wins, top 3 misses, next-week actions.

## Procedure

### Phase 1: Pull the window

Compute ISO week boundary. Pull all your posts in the window via:
```
x-mcp.search_tweets({
  query: "from:<your_handle>",
  start_time: <monday 00:00 UTC>,
  end_time: <sunday 23:59 UTC>,
  max_results: 100
})
```

For each post, fetch its metrics via `get_metrics`.

### Phase 2: Compute baselines

Pull the previous 4 weeks of posts (same query, wider window). Compute medians for:
- Impressions
- Replies
- Likes
- Retweets
- Profile clicks
- Bookmark count (Basic+ tier)

These are your personal baselines. Don't compare against guilyx or anyone else. Compare against past-you.

### Phase 3: Categorize

For each post in the debrief window, mark each metric as `above`, `at`, or `below` baseline. Compute an overall score per post (sum of weighted metric flags; impressions weighted 1x, replies 3x, profile clicks 2x).

Rank posts by overall score. Top 3 = wins. Bottom 3 = misses.

### Phase 4: Pattern-match

Across the wins, identify shared traits:
- Topic cluster (agentic-brain, sanji, building-in-public, replies)
- Format (single post, thread, quote-tweet)
- Hook style (opener pattern)
- Time of day posted
- Day of week

Same for misses.

### Phase 5: Recommend

Generate 2-4 concrete, actionable changes for next week. Examples:
- "Threads about supersede / promote-memory outperformed single posts on the same topics 3x. Default to threads for technical depth this week."
- "Posts at 7-9am ET got 2.4x the impressions of posts at noon ET. Shift the weekly schedule."
- "Posts that mention NotebookLM by name got more replies than generic 'localhost RAG' framings. Use the brand-comparison hook more."
- "Quote-tweets of @user_x landed; standalone posts on the same topics did not. Keep using their threads as the entry point."

Each action is specific enough to execute without re-deciding.

### Phase 6: Write the debrief

`outputs/x-debrief-YYYY-Www.md` with:

```yaml
---
title: X Debrief YYYY-Www
type: debrief
week_start: <ISO>
week_end: <ISO>
posts_in_window: <N>
last_updated: YYYY-MM-DD
---
```

Then sections:
1. **Headline** — three sentences: posts shipped, impressions delta vs last week, profile-clicks delta.
2. **Wins** — top 3 posts with metrics + why.
3. **Misses** — bottom 3 with metrics + why.
4. **Patterns** — what united the wins, what united the misses.
5. **Next week** — the 2-4 recommended actions.

### Phase 7: Echo

Print the headline + next-week actions in the chat. Link to the full file.

## Edge cases

- **No posts in window**: write a single-line debrief: "no posts this week, no signal to debrief."
- **Less than 4 prior weeks of data**: use whatever's available; mark baselines as "early-tenure, treat with caution."
- **Bookmark metric unavailable (Free tier)**: skip that dimension, note in the debrief.
- **Outlier post (1 post got 100x more impressions than the others)**: surface it explicitly. Sometimes the lesson is "do that again." Sometimes it's "lucky algorithm hit, don't over-fit."

## Why the personal baseline

X is a popularity machine. Comparing to bigger accounts will always be discouraging. Comparing to past-you measures whether you're actually improving the craft. The latter is the only signal you can act on.
