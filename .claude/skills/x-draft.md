---
name: x-draft
description: Draft a single X post in your voice. Reads voice/ for tone examples and your recent posts for context.
---

# /x-draft

Draft a single X post in your voice. Reads the `voice/` folder (your best-performing posts) and your recent posts via x-mcp, then drafts a candidate. Diff-and-approve gate before posting.

## When to invoke

- You have a topic but not the prose.
- You want a candidate post you can iterate on, not commit to.
- After `/x-pulse` surfaces a recommendation worth a post.

## What it does

1. Reads every voice file in `voice/` EXCEPT files named `README.md` (any depth).
2. Pulls your last 20 posts via x-mcp `search_tweets`.
3. Drafts a candidate post on the given topic, matching the voice corpus.
4. Shows the draft inside a fenced plain-text block with character count.
5. Prompts for `[a]pprove` / `[e]dit` / `[r]edraft` / `[s]crap`.
6. On approve, appends to today's `outputs/x-drafts-YYYY-MM-DD.md` under `## Approved` and optionally chains to `/x-post`.

## Inputs

- One argument: the topic. Examples:
  - `/x-draft sanji is a localhost replacement for notebooklm`
  - `/x-draft the supersede skill explained`

## Outputs

- Drafted post shown inline.
- Appended to `outputs/x-drafts-YYYY-MM-DD.md` under `## Approved` (on approve) or `## Scrapped` (on scrap).

## Drafts log schema

`outputs/x-drafts-YYYY-MM-DD.md` is the canonical drafts artifact. Schema:

```
---
title: X Drafts YYYY-MM-DD
type: drafts
handle: <@handle>
last_updated: YYYY-MM-DD
---

## Approved

### HH:MM:SS, <topic slug>
chars: <N>
posted: false
post_id:

~~~
<draft text verbatim>
~~~

### HH:MM:SS, <topic slug>
chars: <N>
posted: true
post_id: <id>
post_url: <url>

~~~
<draft text>
~~~

## Scrapped

### HH:MM:SS, <topic slug>
chars: <N>
reason: redraft | scrap | edit-loop

~~~
<original draft>
~~~
```

Notes:
- Each entry's `### HH:MM:SS, <topic slug>` heading uses a comma separator.
- Draft body is fenced with `~~~`, not triple backticks, so a draft containing backticks doesn't break the file.
- `/x-post latest` reads this file, takes the most recent `## Approved` entry with `posted: false`, posts it, then updates `posted: true` + `post_id` + `post_url` in place.

## Procedure

### Phase 1: Load voice

Read every file in `voice/` matching `*.md` or `*.txt`, EXCLUDING any file whose basename is exactly `README.md` at any depth. This rule covers the shipped `voice/README.md` AND any other README the user adds inside `voice/`. Concatenate the rest. Treat as canonical for tone, structure, vocabulary, rhythm.

Also pull your last 20 posts:

```
x-mcp.search_tweets({
  query: "from:<handle> -is:retweet",
  max_results: 20,
})
```

If `voice/` has zero non-README files AND search returns zero posts, prompt the user to seed `voice/` with 5-10 examples before drafting. Do not generate a generic draft.

### Phase 2: Voice analysis

Identify:
- Average char count per post
- Sentence structure pattern
- Opener pattern (e.g., "Folks:", lowercase declaratives, questions)
- Vocab preferences and avoidances
- Punctuation patterns
- Hashtag / mention / link usage

### Phase 3: Draft

Generate ONE candidate post on the given topic that:
- Matches Phase 2 voice signal.
- Stays under 280 chars (Free) or 25,000 chars (Premium; read `config.yaml` `tier: premium` if set, otherwise assume Free).
- Has a clear hook in the first line.
- Avoids vocab the voice analysis flagged as not-yours.

### Phase 4: Present + verify

Show the draft inside a fenced plain-text block. Use `~~~` as the fence delimiter by default. If the draft body contains both `~~~` and triple-backticks, escape the inner `~~~` by inserting a zero-width space at the start of each offending line so the fence isn't terminable by content from the draft.

Below the block:
- `chars: <N> / <limit>`
- `voice match: <one-line note>`

Then prompt:

```
[a]pprove and post / [e]dit / [r]edraft / [s]crap
```

### Phase 5: Branch

- `a` / `approve`: append to `## Approved` in today's drafts log with `posted: false`, then offer `/x-post latest`.
- `e` / `edit`: take user edits, show updated version, re-prompt.
- `r` / `redraft`: regenerate from scratch, present new candidate.
- `s` / `scrap`: append to `## Scrapped` with reason `scrap`, exit.

## Edge cases

- **No voice corpus + no recent posts**: stop and prompt to seed `voice/`.
- **Topic too vague** ("post about ai"): ask for a specific angle.
- **Char count overflow**: tighten once, show diff, ask if user wants further compression or to use `/x-thread`.
- **Draft contains the fence delimiter**: switch fence style or escape per Phase 4.

## Voice training: how to use `voice/`

Drop any of these into `voice/` as plain markdown:
- Your own posts that performed well.
- Posts from accounts whose voice you want to learn from, annotated with `# borrowed-from: @handle`.
- Style notes (e.g., `# rules: no em dashes, lowercase`).

The agent ignores any `README.md` at any depth under `voice/` so convention docs never pollute the corpus.

More is not better. 10-20 strong examples beats 200 mediocre ones.
