# Ingestion Protocol

## Overview

Ingestion converts external material into a standardized raw source file in the wiki's `raw/` directory. Sources are immutable after ingestion.

## Fidelity Requirements (CRITICAL)

These rules apply to ALL ingestion modes. Violation contaminates the raw layer with hallucinations, which silently corrupts every downstream compile, query, and citation.

1. **Verbatim body.** The body of every raw source file must be the source material's own text. No paraphrase, no "summarized for brevity", no filling gaps from general knowledge. If an extraction tool (WebFetch, PDF reader, API proxy) returns obviously truncated, summarized, or model-generated text, STOP and report the truncation — do not silently accept a lossy extraction.

2. **Unknown > guessed.** If a metadata field (author, date, title) is not explicit in the source, write `unknown`. Never infer from the URL slug, the domain, or general knowledge about the site or topic. A wrong date is worse than an unknown date because downstream compile will cite it confidently.

3. **Summary provenance.** Every sentence of the `summary:` frontmatter field must be directly traceable to a passage in the body. If a summary claim can't be cited to the body, remove it.

4. **Tag grounding.** Tags must be derivable from text that appears in the body. Do not add topic tags based on what the source "is probably about" — only for concepts explicitly discussed. When in doubt, fewer tags.

5. **Preserve provenance markers.** Keep every URL, timestamp, quote attribution, figure caption, and citation that appears in the source. These are what downstream compile uses to verify claims.

6. **Declare extraction method.** Add `extraction:` to frontmatter with one of: `webfetch`, `grok-mcp`, `fxtwitter`, `vxtwitter`, `file-read`, `manual-paste`, `pdf-text`, `pdf-ocr`. Downstream compile uses this to decide how much to trust the body.

## Source Types

| Type | Directory | Auto-detect signals |
|------|-----------|-------------------|
| articles | raw/articles/ | General web URLs, blog posts |
| papers | raw/papers/ | arxiv.org, scholar.google, .pdf URLs, academic language |
| repos | raw/repos/ | github.com, gitlab.com URLs |
| notes | raw/notes/ | Freeform text, tweets, no URL |
| data | raw/data/ | .csv, .json, .tsv URLs or files, dataset references |

## URL Ingestion

1. **Detect X.com / Twitter URLs**: If the URL matches `x.com/*/status/*` or `twitter.com/*/status/*`, follow this fallback chain in order:

   **a) Grok MCP (preferred)**: Check if the `grok` MCP server is available by looking for tools matching `mcp__grok__*` (e.g., `mcp__grok__search`). If available, use it to fetch the tweet/thread content. Extract: author handle, display name, full text, date, media descriptions, thread context.
   > Install: [github.com/nvk/ask-grok-mcp](https://github.com/nvk/ask-grok-mcp)

   **b) FxTwitter proxy**: If Grok MCP is not available, rewrite the URL:
   - `x.com/user/status/123` → `https://api.fxtwitter.com/user/status/123`
   - WebFetch this API URL — it returns JSON with full tweet text, author, media, and thread data.
   - Parse the JSON response for `tweet.text`, `tweet.author`, `tweet.created_at`.

   **c) VxTwitter proxy**: If FxTwitter fails, try:
   - `x.com/user/status/123` → `https://api.vxtwitter.com/user/status/123`
   - Same JSON extraction as FxTwitter.

   **d) Direct WebFetch**: Last resort — WebFetch the original `x.com` URL. This often returns limited content (login walls), but sometimes works for public tweets.

   **e) Manual fallback**: If all above fail, report: "Could not fetch tweet content. Options: install [ask-grok-mcp](https://github.com/nvk/ask-grok-mcp) for X.com access, or paste the tweet text manually via `/wiki:ingest \"text\" --title \"@author tweet\"`."

   Type: notes (unless overridden).

2. **General URLs**: Use WebFetch to retrieve content. Prompt:

   > "Extract the article from this page VERBATIM. Return the complete article text exactly as it appears in the source — do NOT summarize, paraphrase, or 'clean up' the content. Preserve every factual claim, number, quote, attribution, and citation. Fields: title (as printed), author(s) (from byline only, else 'unknown' — never infer), date published (as printed, else 'unknown' — never infer), full article body text. If the page is gated, paywalled, rate-limited, or returns only partial content, say so EXPLICITLY at the top of the response. Format as clean markdown, preserving headings, lists, block quotes, and inline links."

3. **GitHub repo URLs**: Use WebFetch with prompt:

   > "Extract from this GitHub repository VERBATIM: repo name, description (as shown on the repo page, not inferred), primary language(s) from the language bar (not guessed from the name), stated purpose (README intro only), and the full README content. If any field is absent, return 'unknown' — do not infer from the repo name or URL. Format as clean markdown."

4. **Failure handling**: If WebFetch fails (auth wall, paywall), report the failure. Suggest: paste content manually via `/wiki:ingest "text" --title "Title"`.

## File Ingestion

1. Read the file directly
2. Markdown → preserve formatting
3. Plain text → wrap in markdown
4. JSON/CSV/structured data → describe schema + representative sample (not full dataset)
5. Images → create a metadata stub noting the image path and any visible content description

## Freeform Text Ingestion

1. User provides quoted text as the argument
2. If `--title` not provided, derive a title from the first sentence or ask
3. Auto-tag based on content keywords

## Inbox Processing

The `inbox/` directory is a drop zone. Users dump files there via Finder, `cp`, etc.

### Processing `--inbox`:

1. Scan `inbox/` for all files (exclude `.processed/` subdirectory and hidden files)
2. For each file:
   - `.url` or `.webloc` files → extract the URL, then follow URL ingestion flow
   - `.md` or `.txt` files → ingest as notes or articles (auto-detect)
   - `.pdf` files → create a metadata stub, note the file path for reference
   - `.json`, `.csv`, `.tsv` → ingest as data
   - Other files → create a metadata stub noting file type and path
3. Move each processed file to `inbox/.processed/` (or delete if user did not pass `--keep`)
4. Report each item processed
5. If 5+ items were processed, suggest: "You've ingested N new sources. Want me to compile? Run `/wiki:compile`"

## Slug Generation

1. Take the title, lowercase, replace spaces with hyphens, remove special characters
2. Prepend today's date: `YYYY-MM-DD-`
3. Truncate to 60 characters max (not counting .md extension)
4. Example: "Attention Is All You Need" → `2026-04-04-attention-is-all-you-need.md`
5. If a file with that slug already exists, append `-2`, `-3`, etc.

## Verification Pass (REQUIRED before index updates)

After writing the raw source file but BEFORE any index updates, perform this pass. It is the last line of defense against hallucination contamination.

1. **Re-read the file you just wrote** — don't work from memory.
2. **Check each frontmatter field against the body**:
   - `title` — matches the source's own printed title, or a clearly-derived variant?
   - `author` / `date_published` — present as byline/dateline in the body? If not, the field must be `unknown`.
   - Each sentence of `summary` — can you quote the body passage that supports it?
   - Each `tag` — can you point to body text that explicitly discusses that concept?
3. **Scan the body for fabrication signals**:
   - Sentences that sound more confident or polished than the rest of the source
   - Specific numbers, dates, or names that appear only once and aren't sourced elsewhere in the body
   - Unattributed opinions, interpretations, or "takeaway" paragraphs not present in the original
   - Abrupt topic jumps that suggest the extraction tool stitched unrelated content
4. **If any check fails**: revise the file — remove ungrounded claims from `summary`/`tags`, change unverifiable metadata to `unknown`, or re-extract the body if extraction was lossy. If the source cannot be verified (e.g., extraction returned boilerplate or was clearly summarized), DELETE the file and report to the user — do NOT proceed to index updates.
5. **On pass**: set `verification: passed` in the frontmatter and append to `log.md`: `## [YYYY-MM-DD] verify | <slug> — OK`. Note any fields set to `unknown` in the log.

## Post-Ingestion Index Updates

Only run after the Verification Pass has passed. Update indexes in order:

1. `raw/{type}/_index.md` — add row to Contents table
2. `raw/_index.md` — add row to Contents table
3. `_index.md` (master) — increment source count, add to Recent Changes

## Batch Ingestion

If the user provides multiple URLs or paths (comma-separated, space-separated, or one per line), process each sequentially. Report progress after each item.

## Compilation Nudge

After ingestion, count uncompiled sources (sources ingested after last compile date). If 5+, suggest running `/wiki:compile`.
