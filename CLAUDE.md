# x-advisor

A Claude Code agent that runs your X (Twitter) workflow: daily pulse, post drafts in your voice, reply-target hunting, thread drafting, and weekly engagement debriefs.

Backed by the [x-mcp](https://github.com/INFATOSHI/x-mcp) MCP server (set up separately; instructions in `README.md`).

## Quick map

```
.claude/skills/    six skills you can invoke as slash commands
.claude/commands/  command aliases mirroring the skills
voice/             your voice training corpus (start here)
outputs/           generated artifacts (pulses, drafts, threads, debriefs)
niche.yaml         optional account allowlist for /x-targets
README.md          public-facing pitch + setup
```

## The skills

| Command | What it does |
|---|---|
| `/x-pulse` | Daily timeline + my-analytics digest. Reads x-mcp `get_timeline`, `get_mentions`, `get_metrics`. Outputs a one-page brief. |
| `/x-draft <topic>` | Drafts a single post in your voice on a topic. Reads `voice/` for tone. Diff-and-approve gate. |
| `/x-targets [max_age_hours] [max_results]` | Surfaces high-velocity posts from your niche worth a manual reply. Ranked by engagement / age. |
| `/x-post <draft>` | Ships a drafted post via x-mcp `post_tweet`. Typed `post` confirmation required. |
| `/x-thread <topic> [--length N]` | Drafts a multi-post thread. Posting is manual paste (see thread skill for the API caveat). |
| `/x-debrief [YYYY-Www]` | Weekly engagement review. Wins, misses, patterns, 2-4 next-week actions. |

Each skill prompt lives at `.claude/skills/<name>.md`. Read the source. Fork what you need.

## Voice

The agent's drafting quality is gated by what's in `voice/`. Seed it with 10-20 of your best-performing posts before you run `/x-draft` or `/x-thread`. See `voice/README.md` for the convention.

## Outputs

Everything the agent generates lands in `outputs/` with a dated filename:

- `outputs/x-pulse-YYYY-MM-DD.md` — daily briefs
- `outputs/x-drafts-YYYY-MM-DD.md` — drafted posts (approved + scrapped)
- `outputs/x-targets-YYYY-MM-DD-HH.md` — target lists
- `outputs/x-threads-YYYY-MM-DD.md` — drafted threads
- `outputs/x-posted-YYYY-MM-DD.md` — what was actually shipped
- `outputs/x-debrief-YYYY-Www.md` — weekly debriefs

`outputs/` is in `.gitignore` so your drafts and analytics stay local.

## API constraints worth knowing

x-mcp wraps the X API. The API has progressively restricted automated clients through 2025-2026:

- **Programmatic replies restricted** (Feb 2026): the `reply_to_tweet` tool only works if the original poster @mentioned or quoted you. Affects all self-serve tiers. Workaround: `quote_tweet`, or reply manually with the targets from `/x-targets`.
- **Likes removed from Free tier** (Aug 2025): like via API only works on Basic+ ($200/mo).
- **Bookmarks** require Basic+.
- **Post volume**: 500/mo Free, 10k/mo Basic.

The advisor is designed around these constraints. It surfaces reply targets but doesn't reply for you — that's where Premium reply-boost on your account does the work.

## Daily rhythm

```
morning  → /x-pulse → read the brief, decide focus for the day
mid-day  → /x-targets → spend 30-60 min on manual replies to ranked targets
ad-hoc   → /x-draft "<topic>" → /x-post latest, when a post idea hits
sunday   → /x-debrief → review the week, adjust next week's focus
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
