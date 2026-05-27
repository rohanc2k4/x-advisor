# x-advisor

A Claude Code agent that runs your X (Twitter) workflow: daily pulse, post drafts in your voice, reply-target hunting, thread drafting, and weekly engagement debriefs.

Backed by [rohanc2k4/x-mcp](https://github.com/rohanc2k4/x-mcp), a forked + patched MCP server (set up separately; instructions in `README.md`).

## Quick map

```
.claude/skills/      six skills you can invoke as slash commands
.claude/commands/    command aliases mirroring the skills
voice/               your voice training corpus (start here)
outputs/             generated artifacts (pulses, drafts, threads, debriefs)
config.yaml          your handle + timezone + optional cached user_id (gitignored)
niche.yaml           optional account allowlist for /x-targets (gitignored)
.cache/              resolved user_id and other ephemeral state (gitignored)
README.md            public-facing pitch + setup
```

## The skills

| Command | What it does |
|---|---|
| `/x-pulse` | Daily timeline + my-analytics digest. Calls `get_user` (cached), `get_mentions`, `search_tweets` (per niche handle), `get_metrics` (your posts). Produces a one-page brief. |
| `/x-draft <topic>` | Drafts a single post in your voice on a topic. Reads `voice/` (excluding any `README.md` at any depth) for tone. Diff-and-approve gate. |
| `/x-targets [max_age_hours] [max_results]` | Surfaces high-velocity posts from your niche worth a manual reply. Ranks by `public_metrics` from `search_tweets` (no `get_metrics` calls on others' posts; that endpoint is author-only for non-public fields). |
| `/x-post <draft \| latest>` | Ships a drafted post via `post_tweet`. Per-post 4-char hex hash-code confirmation required (case-insensitive). |
| `/x-thread <topic> [--length N]` | Drafts a multi-post thread. Posting is manual paste only (Feb 2026 self-reply restrictions). |
| `/x-debrief [YYYY-Www]` | Weekly engagement review. Wins, misses, patterns, 2-4 next-week actions. ISO week boundaries computed in your configured timezone. |

Each skill prompt lives at `.claude/skills/<name>.md`. Read the source. Fork what you need.

## Voice

The agent's drafting quality is gated by what's in `voice/`. Seed it with 10-20 of your best-performing posts before you run `/x-draft` or `/x-thread`. See `voice/README.md` for the convention. The voice loader explicitly excludes any `README.md` at any depth under `voice/` so docs never pollute the corpus.

## Outputs

Everything the agent generates lands in `outputs/` with a dated filename:

- `outputs/x-pulse-YYYY-MM-DD.md` (daily briefs)
- `outputs/x-drafts-YYYY-MM-DD.md` (drafted posts under `## Approved` and `## Scrapped` sections; `/x-post latest` reads from here)
- `outputs/x-targets-YYYY-MM-DD-HHMM.md` (target lists)
- `outputs/x-threads-YYYY-MM-DD.md` (drafted threads + paste-ready blocks)
- `outputs/x-posted-YYYY-MM-DD.md` (what was actually shipped)
- `outputs/x-debrief-YYYY-Www.md` (weekly debriefs; week-year is ISO, not calendar)

`outputs/` is gitignored so your drafts and analytics stay local.

## API constraints worth knowing

The X API has progressively restricted automated clients through 2025-2026:

- Programmatic replies restricted (Feb 2026): `reply_to_tweet` only succeeds if the original poster @mentioned or quoted you. Affects all self-serve tiers. The agent does not use this tool; it surfaces reply targets and you reply manually.
- Likes removed from Free tier (Aug 2025): `like_tweet` requires Basic+. The agent doesn't depend on it.
- Bookmarks require Basic+.
- Post volume: 500/mo Free, 10k/mo Basic.
- `search_tweets` accepts `query`, `max_results`, `next_token`. It does NOT accept `since_id`, `start_time`, or `end_time`. Time windows are computed by filtering `created_at` locally on returned tweets.
- `get_metrics` returns full `non_public_metrics` / `organic_metrics` only for posts you authored. For others' posts, use the `public_metrics` field returned in `search_tweets` responses.
- `get_timeline` and `get_following` require a numeric `user_id` (resolve via `get_user` first; cached in `.cache/me.json`).

The advisor is designed around these constraints.

## Daily rhythm

```
morning  -> /x-pulse    -> read the brief, decide focus for the day
mid-day  -> /x-targets  -> spend 30-60 min on manual replies to ranked targets
ad-hoc   -> /x-draft "<topic>" -> /x-post latest, when a post idea hits
sunday   -> /x-debrief  -> review the week, adjust next week's focus
```

## Adding new skills

Same convention as the rest of my framework ecosystem ([agentic-brain](https://github.com/rohanc2k4/agentic-brain) ships the methodology):

1. Brainstorm conversationally first.
2. Write the spec at `docs/superpowers/specs/YYYY-MM-DD-<name>-design.md` (create the dir if missing).
3. Write the plan at `docs/superpowers/plans/YYYY-MM-DD-<name>.md`.
4. Build the skill at `.claude/skills/<name>.md`.
5. Mirror the command at `.claude/commands/<name>.md`.
6. Update the skill table in this file and `README.md`.

Skills should formalize after the third time you asked Claude to do the same thing. Until then, do it conversationally.
