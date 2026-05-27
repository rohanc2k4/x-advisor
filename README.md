# x-advisor

A [Claude Code](https://claude.com/claude-code) agent that runs your X (Twitter) workflow.

Daily pulse on your timeline and analytics. Post and thread drafts in your voice. Reply-target hunting in your niche. Weekly engagement debriefs. All driven by six small skills you can read, fork, and rewrite.

Backed by a [forked + patched x-mcp](https://github.com/rohanc2k4/x-mcp) for the X API plumbing. The fork removes a hidden post-text mutation in upstream that appended a promotional disclosure to every outgoing tweet.

## The skills

| Command | What it does |
|---|---|
| `/x-pulse` | Morning brief from your timeline, mentions, and last 10 posts' metrics |
| `/x-draft <topic>` | Drafts a single post in your voice. Diff-and-approve gate |
| `/x-targets` | Surfaces fresh, high-velocity posts in your niche worth a manual reply |
| `/x-post <draft>` | Ships a drafted post via the MCP. Per-post hash-code confirmation required |
| `/x-thread <topic>` | Drafts a multi-post thread. Manual paste to ship; programmatic posting not supported (Feb 2026 self-reply restrictions) |
| `/x-debrief` | Sunday review: wins, misses, patterns, next-week actions |

## Why this exists

Twitter growth for builders is mechanical: post in your niche consistently, reply early on big-account posts, debrief weekly, iterate. The mechanical parts (pulling analytics, drafting in your voice, finding fresh reply targets, reviewing week-over-week) eat 2-4 hours per week if you do them by hand. This agent compresses it to 20-30 minutes a day.

It does NOT auto-reply, auto-like, or auto-engage. The Feb 2026 X API restrictions made programmatic replies unreliable on all self-serve tiers, and the worthwhile audience comes from your actual voice on the actual replies anyway. The agent puts you in front of the right targets at the right time. You write the reply.

## Setup

### 1. Set up the x-mcp fork

The agent calls the X API through [rohanc2k4/x-mcp](https://github.com/rohanc2k4/x-mcp), a fork of Infatoshi/x-mcp with one patch: upstream hardcoded a `[<model> on behalf of @elliotarledge]` suffix on every outgoing tweet. The fork removes it. Tweets ship verbatim.

To set up:

```bash
git clone https://github.com/rohanc2k4/x-mcp.git
cd x-mcp
npm install
npm run build
```

Then get five X API credentials from the [X Developer Portal](https://developer.x.com/en/portal/dashboard) and wire the MCP into Claude Code's config (see x-mcp's README for the exact config block). Make sure your access token has write permission (otherwise posting fails with a 403; see x-mcp's troubleshooting section).

### 2. Clone this repo

```bash
git clone https://github.com/rohanc2k4/x-advisor.git
cd x-advisor
```

### 3. Create `config.yaml`

```bash
cp config.example.yaml config.yaml
```

Edit `config.yaml`. Set `handle` (your X handle without the leading `@`) and `timezone` (IANA, e.g. `America/New_York`). The agent caches your numeric `user_id` on first run.

### 4. Seed `voice/`

Drop 10-20 of your best X posts into `voice/` as plain markdown. See `voice/README.md`. Without this, `/x-draft` and `/x-thread` have no voice signal and the agent refuses to draft generic copy.

### 5. Optional: write `niche.yaml`

`/x-targets` searches a set of in-niche accounts. By default it uses everyone you follow (capped at 100). For sharper targeting, list the accounts that matter:

```yaml
niche:
  - handle: simonw
    tag: ai-tooling
  - handle: jasonzhou
    tag: ai-agents
  - handle: theo
    tag: ai-tooling
```

### 6. Use it

Open Claude Code in this repo. Try:

```
/x-pulse
```

That's the daily kickoff. Then read the brief, run `/x-targets`, draft a post or two with `/x-draft`, ship one with `/x-post`, and run `/x-debrief` on Sunday.

## Daily rhythm

```
morning   -> /x-pulse    -> read the brief, set focus for the day
mid-day   -> /x-targets  -> 30-60 min on manual replies to the top of the list
ad-hoc    -> /x-draft "<topic>" -> /x-post latest, when an idea hits
sunday    -> /x-debrief  -> assess the week, adjust next week's plan
```

## API constraints worth knowing

x-mcp wraps the X API. The X API has progressively restricted automated clients through 2025-2026:

- **Programmatic replies** restricted on all self-serve tiers (Feb 2026). The MCP can only reply if the original poster @mentioned or quoted you. The agent does not try to. It surfaces reply targets; you reply manually with your account's Premium reply-boost doing the visibility work.
- **Programmatic likes** removed from Free tier (Aug 2025). Basic+ tier is required for `like_tweet`. The agent doesn't depend on it.
- **Post volume**: Free tier caps at 500/month. More than enough for daily posting.
- **Bookmarks** need Basic+ tier. The agent skips bookmark features on Free.

## Design principles

- **You stay in the loop**: nothing posts without typing a per-post hash code displayed alongside the draft. The friction is the point.
- **Voice is gated by `voice/`**: drafts are only as good as the corpus. Curate it.
- **One file per skill**: read `.claude/skills/*.md` to see exactly what each skill does.
- **Outputs are local**: everything the agent generates lands in `outputs/`, which is gitignored. Your drafts and analytics never leave your machine.
- **Personal baseline**: `/x-debrief` compares you against past-you, not against bigger accounts. The only signal you can act on.

## Related

- [agentic-brain](https://github.com/rohanc2k4/agentic-brain), the general-purpose Claude Code framework this agent's skill conventions came from
- [rohanc2k4/x-mcp](https://github.com/rohanc2k4/x-mcp), the patched MCP this agent calls into
- [Infatoshi/x-mcp](https://github.com/Infatoshi/x-mcp), upstream MCP this repo forks from

## License

MIT.
