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

1. Pulls your home timeline via the x-mcp `get_timeline` tool with `count: 50`.
2. Pulls your unread mentions via `get_mentions` (paginated until you hit 24h ago).
3. Pulls metrics for your last 10 posts via `get_metrics`.
4. Synthesizes a one-page brief:
   - **Top of feed**: 5 most-engaged posts on your timeline in the last 24h, with one-line on why each landed.
   - **Mentions**: every account that @mentioned you in the window, what they said, whether it needs a reply.
   - **Your post performance**: each of your last 10 posts with impressions, replies, retweets, profile clicks, and a one-line read on it.
   - **Today's recommendation**: 1-3 concrete actions (reply target, post topic, thread idea) based on what's hot right now.
5. Writes the brief to `outputs/x-pulse-YYYY-MM-DD.md`.

## Inputs

- None. Reads live from x-mcp.

## Outputs

- `outputs/x-pulse-YYYY-MM-DD.md` — the brief.
- A concise summary printed in the chat with the brief's top recommendation, so you can act without opening the file.

## Procedure

### Phase 1: Pull live data

Call x-mcp tools in parallel where possible:
- `get_timeline` with `count: 50` and `since_id` from the last `x-pulse-*.md` file (if one exists) — only fetch new content.
- `get_mentions` with `since_id` from the last brief.
- `get_metrics` for the IDs of your last 10 posts (look them up via `search_tweets` with `from:<your_handle>`).

If a tool returns a rate-limit error, note the reset timestamp in the brief and bail on that section gracefully.

### Phase 2: Synthesize

For each section:
- **Top of feed** — pick the 5 with highest engagement velocity (likes + replies + retweets divided by post age in hours). Skip posts from accounts you don't follow in your niche unless the post is breaking news.
- **Mentions** — group by author. For each, summarize the @mention in one line. Mark each as `reply` / `like` / `skip` based on whether it deserves engagement.
- **Your post performance** — for each post, compare to your last 30-day median for that metric. Mark as `above`, `at`, or `below`.

### Phase 3: Recommend

Generate 1-3 concrete actions for the day. Each action is a single sentence with a verb. Examples:
- "Reply to @user_x's post about agent design (linked above) — they have 50k followers in your niche and the post is 12 minutes old."
- "Draft a thread about the supersede skill — three posts on your feed in the last hour are talking about wiki-rot in LLM contexts."

### Phase 4: Write the brief

Write to `outputs/x-pulse-YYYY-MM-DD.md` with this frontmatter:

```yaml
---
title: X Pulse YYYY-MM-DD
type: pulse
since: <ISO of previous brief or 24h ago>
until: <now ISO>
last_updated: YYYY-MM-DD
---
```

Then four sections matching the synthesis above.

### Phase 5: Echo the headline

Print the today's-recommendation section in the chat, plus the path to the full brief.

## Edge cases

- **First run, no previous brief**: use 24h ago as `since`.
- **Rate limited**: write what you have, mark missing sections with `[rate-limited, retry after <ISO>]`.
- **Empty timeline**: bail out with a one-liner brief explaining no new activity.
- **Mentions endpoint returns nothing**: that's fine, write "no new mentions" in the brief.
