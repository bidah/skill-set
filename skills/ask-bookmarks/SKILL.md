---
name: ask-bookmarks
description: Answer a question using the user's own X/Twitter bookmarks via the local `ft` CLI. Runs several full-text searches with query variations, ranks the matching bookmarks, synthesizes an answer, and ends with a References section of tweet links. Use when the user asks a question and wants it answered from their saved bookmarks, says "search my bookmarks", "what did I bookmark about X", "find the tweet about X", or invokes /ask-bookmarks.
---

# Ask Bookmarks

Answer the user's question from their X/Twitter bookmarks stored locally by the `ft` CLI, then cite the source tweets.

## Workflow

1. **Extract intent.** Pull the key concepts from the question. List 3–6 search angles: the literal terms, synonyms, related jargon, author handles, and product/tool names.

2. **Run several searches.** For each angle, query the local DB as JSON:
   ```bash
   ft list --query "<FTS5 query>" --json --limit 15
   ```
   - FTS5 syntax: space = AND, `OR`, `NOT`, `"exact phrase"`. Use `OR` to broaden (`"claude code" OR "coding agent"`), quotes for multi-word phrases.
   - Run a mix: one narrow phrase query, one broad OR query, plus per-synonym queries. Prefer many small queries over one giant one.
   - Optional filters: `--author <handle>`, `--after YYYY-MM-DD`, `--before YYYY-MM-DD`.

3. **Merge & rank.** Combine results, dedupe by `id`. Rank by relevance to the question first, then by engagement (`bookmarkCount`, `likeCount`) and recency (`postedAt`) as tie-breakers. Drop obvious non-matches.

4. **Read closely if needed.** The JSON `text` is often truncated mid-tweet. For the top candidates, get the full text:
   ```bash
   ft show <id> --json
   ```

5. **Answer.** Write a direct answer grounded ONLY in what the bookmarks say. Attribute claims to authors inline (e.g. "@handle notes …"). If the bookmarks don't cover the question, say so plainly — don't pad with outside knowledge unless the user asks.

6. **Always end with References.** Use the `url` field from each cited bookmark verbatim:
   ```
   ## References
   - [@authorHandle — short description](https://x.com/authorHandle/status/<id>)
   ```
   Only list tweets actually used in the answer. Number-match them to inline citations if you used `[1]`-style markers.

## Notes
- `ft search <query>` is the human-readable variant; `ft list --query ... --json` is preferred here because it's structured and exposes `url`, `id`, and engagement counts.
- Bookmarks are unclassified by default, so `--category`/`--domain` filters are usually empty — rely on full-text search.
- If a search errors on FTS5 syntax (e.g. a stray special char), retry with the terms quoted as an exact phrase.
- If zero results across all angles, broaden terms and try again before concluding nothing is bookmarked.
