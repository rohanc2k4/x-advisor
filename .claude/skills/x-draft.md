---
name: x-draft
description: Draft a single X post in your voice on a given topic. Reads voice/ for tone examples and your recent posts for context.
---

# /x-draft

Draft a single X post in your voice. The skill reads the `voice/` folder (your best-performing posts) and your recent posts via x-mcp, then drafts a candidate. Diff-and-approve gate before posting.

## When to invoke

- You have a topic but not the prose.
- You want a candidate post you can iterate on, not commit to.
- After `/x-pulse` surfaces a recommendation worth a post.

## What it does

1. Reads every `.md` and `.txt` file in `voice/` (your voice training corpus).
2. Pulls your last 20 posts via x-mcp `search_tweets` with `from:<your_handle>`.
3. Drafts a candidate post on the given topic, matching the voice corpus.
4. Shows the draft with a character count and asks for approval, edits, or scrap.
5. On approval, optionally chains into `/x-post` to ship.

## Inputs

- One argument: the topic. Examples:
  - `/x-draft sanji is a localhost replacement for notebooklm`
  - `/x-draft the supersede skill explained`
  - `/x-draft my honest take on contextual retrieval for student vaults`

## Outputs

- A drafted post (with char count) shown inline.
- On approval: saved to `outputs/x-drafts-YYYY-MM-DD.md` as a dated draft log AND optionally posted via `/x-post`.

## Procedure

### Phase 1: Load voice

Read every file in `voice/` (skip `.gitkeep`). Concatenate. Treat as the canonical reference for tone, structure, vocabulary, and rhythm.

Also pull your last 20 posts via:
```
x-mcp.search_tweets({ query: "from:<your_handle> -is:retweet", max_results: 20 })
```

If `voice/` is empty, fall back entirely to the recent posts as the voice signal.

### Phase 2: Voice analysis

Identify:
- Average char count per post
- Sentence structure pattern (one-line declarative? multi-sentence? bullet-style?)
- Opener pattern (e.g., "Folks:", lowercase declarations, questions)
- Vocab preferences (what words you use; what words you don't)
- Punctuation patterns
- Whether you use hashtags / mentions / links

### Phase 3: Draft

Generate ONE candidate post on the given topic that:
- Matches the voice analysis from Phase 2.
- Stays under 280 chars (or 25,000 if Premium — confirm with user once at setup).
- Has a clear hook in the first line.
- Avoids any vocabulary the voice analysis flagged as not-yours.

### Phase 4: Present

Show the draft inside a fenced plain-text code block (so the user can copy verbatim). Below it, show:
- Char count
- Voice match notes (1-3 lines: "matches your opener pattern, avoids the words X/Y/Z, char count is in your usual range")

Then prompt:
```
[a]pprove and post / [e]dit / [r]edraft / [s]crap
```

### Phase 5: Branch

- `approve` → save to `outputs/x-drafts-YYYY-MM-DD.md` and chain to `/x-post` for confirmation + posting.
- `edit` → take user edits, show updated version, re-prompt.
- `redraft` → regenerate from scratch, present new candidate.
- `scrap` → save to `outputs/x-drafts-YYYY-MM-DD.md` under a `## Scrapped` section for later review, then exit.

## Edge cases

- **No voice corpus and no posts**: ask the user to seed `voice/` with 5-10 of their favorite posts before drafting. Explain that the voice match is the whole point.
- **Topic too vague** ("post about ai"): ask for one specific angle before drafting.
- **Char count overflow**: tighten one round, present, ask if user wants further compression or to ship a thread instead via `/x-thread`.

## Voice training: how to use `voice/`

Drop any of these into `voice/` as plain markdown:
- Your own posts that performed well (copy from X)
- Posts from accounts whose voice you want to learn from (annotated with `# borrowed-from: @handle`)
- Style notes (e.g., `# rules: no em dashes, lowercase, no hashtags`)

More is not better. 10-20 strong examples beats 200 mediocre ones.
