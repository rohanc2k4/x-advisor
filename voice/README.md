# voice/

This is where you teach the agent how you sound.

## What to put here

- Your own X posts that performed well, one per file or in a single `my-posts.md`.
- Posts from accounts whose voice you want to borrow from, annotated with `# borrowed-from: @handle` at the top.
- Style rules you want enforced (e.g., `# rules: no em dashes, lowercase, no hashtags`).

## What NOT to put here

- Long-form articles, tweets unrelated to your style, random content. Noise hurts voice match.
- Spammy or low-quality posts, even your own. The agent will treat anything here as a template.

## How much

10-20 strong examples beats 200 mediocre ones. The voice signal is the median of what's here, not the maximum.

## When to update

- After a post performs surprisingly well: add it.
- After a post you regret posting: don't add it (and remove anything similar already here).
- Monthly: prune the bottom third.

## File format

Plain markdown or plain text. The agent reads every `.md` and `.txt` file in this directory EXCEPT any file named `README.md` (at any depth). This file is documentation; it's explicitly excluded from the corpus.

Example file (`my-posts.md`):

```
# my voice, pinned examples

---

Folks: my second brain runs on claude code.

a vault of wikilinked .md files. concurrent memory that keeps learning, powered by skills like drift, crystallize, supersede, and promote-memory.

make your day more efficient. agentic-brain on github.

---

# rules
- no em dashes
- lowercase
- short declarative sentences
- "Folks:" opener for build-in-public posts
```
