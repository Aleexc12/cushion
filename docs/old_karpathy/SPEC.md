# Knowledge Base: High-Level Spec

## What this is

A personal knowledge base feature for Cushion, inspired by [Karpathy's LLM Wiki](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f) and [Cole Medin's claude-memory-compiler](https://github.com/coleam00/claude-memory-compiler). The user drops files into a `sources/` folder, clicks "ingest," and the LLM compiles structured, cross-referenced knowledge articles into a `knowledge/` folder. No RAG, no vector DB — just an `index.md` the LLM reads to find what it needs.

Karpathy's approach is powerful but requires discipline (CLI commands, writing a CLAUDE.md schema, manually running ingest/lint). Cushion makes it a button.

## Folder structure

```
vault/
  sources/          # User-owned. Drop anything here: md, pdf, txt, images, etc.
  knowledge/        # LLM-owned. Structured wiki articles.
    index.md        # Master catalog — every article with a one-line summary
    log.md          # Append-only chronological record of ingests, queries, lints
    concepts/       # Atomic knowledge articles (one topic per file)
    connections/    # Cross-cutting insights linking 2+ concepts
    qa/             # Filed-back query answers
```

- `sources/` is immutable from the LLM's perspective — it reads but never modifies.
- `knowledge/` is LLM-owned. The user can read and manually edit, but the LLM is the primary author.
- Both folders live at the vault root, visible in the normal file tree. No special UI treatment — they're just folders.

## Core operations

### 1. Ingest

**Trigger:** User clicks an "ingest" button in the UI.

**Behavior:**
- First run: processes everything in `sources/`.
- Subsequent runs: incremental — only new files and files changed since last ingest (tracked via SHA-256 hashes in a state file).
- For each source file, the LLM:
  1. Reads the source content (text extraction for PDFs, etc. — OpenCode handles this).
  2. Reads `knowledge/index.md` to understand what already exists.
  3. Reads existing articles that the source might touch.
  4. Creates new `concepts/` articles for new topics.
  5. Updates existing articles if the source adds information.
  6. Creates `connections/` articles when non-obvious relationships emerge.
  7. Updates `index.md` with new/modified entries (summary + source + date).
  8. Appends to `log.md`.
- Runs via OpenCode (spawn a session dedicated to the ingest task).
- When a new source contradicts an existing knowledge page, the agent silently updates the page. Newer source wins.
- When finished, shows a dialog: "Ingestion complete. Created X articles, updated Y."

**Article format** (adapted from cole-medin):

```markdown
---
title: "Concept Name"
aliases: [alternate-name]
tags: [domain, topic]
sources:
  - "sources/filename.pdf"
  - "sources/other.md"
created: 2026-04-07
updated: 2026-04-07
---

# Concept Name

**Summary:** One sentence. (This is what index.md uses for retrieval.)

## Key Points

- Bullet points, each self-contained

## Details

Encyclopedia-style paragraphs.

## Related Concepts

- [[concepts/related-concept]] - How it connects

## Sources

- [[sources/filename.pdf]] - What was extracted from this source
```

**Index format:**

```markdown
# Knowledge Base Index

| Article | Summary | Sources | Updated |
|---------|---------|---------|---------|
| [[concepts/topic-name]] | One-line description | sources/file.pdf | 2026-04-07 |
```

### 2. Query

**Trigger:** User asks a question in the chat sidebar.

**Behavior:**
- The agent reads `knowledge/index.md` first.
- Picks 3-10 relevant articles based on summaries.
- Reads those articles in full.
- Synthesizes an answer with `[[wikilink]]` citations.
- Optionally files the answer back as a `knowledge/qa/` article (compounding loop — every question makes the KB smarter).

This uses the existing chat sidebar and OpenCode agent — no new UI surface needed. The agent just has access to the knowledge base as part of the vault.

### 3. Lint

**Trigger:** User clicks a "lint" button, or runs automatically after ingest.

**Checks:**
1. **Broken links** — `[[wikilinks]]` pointing to non-existent articles.
2. **Orphan pages** — Articles with zero inbound links.
3. **Orphan sources** — Source files that haven't been ingested.
4. **Stale articles** — Source files changed since the article was last compiled.
5. **Missing backlinks** — A links to B but B doesn't link back.
6. **Sparse articles** — Below a word count threshold.
7. **Contradictions** — Conflicting claims across articles (LLM-powered, costs tokens).

Outputs a report. Structural checks (1-6) are free. Contradiction check (7) requires an LLM call.

### 4. Backlinks tool

Ship independently of the wiki feature — valuable on its own.

- Parse `[[wikilinks]]` and `[text](file.md)` across the entire vault.
- Build an inverted index.
- Expose `get_backlinks(file)` and `get_outgoing_links(file)` as agent tools.
- Works across both `sources/` (as link targets only) and `knowledge/` (full bidirectional).

### 5. Provenance and freshness

Each knowledge article tracks which source files contributed to it (in frontmatter `sources:` field). This enables:

- **Staleness detection:** If a source file changes after an article was compiled from it, the article is stale.
- **Provenance hover:** UI can show "compiled from X, Y, Z" for any knowledge article.
- **Freshness badges:** "Not re-checked in N ingests" indicators.
- **Source → article tracing:** Given a source file, find all knowledge articles derived from it.

### 6. State tracking

A state file (JSON, gitignored) tracks:
- Map of source filenames to SHA-256 hashes and last-ingested timestamps.
- Total ingest count.
- Last lint timestamp.

This enables incremental ingest (skip unchanged files) and stale article detection.

## How the LLM retrieves knowledge (no RAG)

At personal scale (50-500 articles), the LLM reading `index.md` outperforms vector similarity search. The LLM understands intent ("what auth patterns do I use?") while cosine similarity just matches words. RAG becomes necessary only at ~2,000+ articles when the index exceeds the context window.

The retrieval flow:
1. LLM reads `index.md` (article summaries in a table).
2. Selects 3-10 relevant articles by reasoning over the summaries.
3. Reads those articles in full.
4. Synthesizes an answer.

The **Summary** line in each article and in the index table is what makes this work. It's the only thing the LLM reads before deciding whether to open the full article.

## What Cushion provides that Obsidian + CLI can't

- **One-click ingest** instead of CLI commands and CLAUDE.md schemas.
- **Integrated query** via the chat sidebar — no separate tool.
- **Provenance UI** — hover to see what sources built an article, badge stale pages.
- **Editor signals** — Cushion knows when files change, enabling incremental processing and staleness tracking without the user running commands.
- **No setup** — the `sources/` and `knowledge/` folders, article formats, and index structure are all handled by the app. The user just drops files and clicks ingest.

## Build phases

1. **Ingest pipeline** — the core: source reading, compile prompt, article generation, index maintenance, incremental state tracking. UI: ingest button + completion dialog.
2. **Query integration** — agent uses `index.md` for retrieval when answering questions. File-back for Q&A articles.
3. **Lint** — structural checks + LLM contradiction detection. UI: lint button + report display.
4. **Provenance and freshness** — source tracking in frontmatter, staleness detection, UI indicators (hovers, badges).
5. **State and polish** — state.json management, error handling, progress indicators during ingest, edge cases (empty sources, huge files, ingest interruption).

### Future / maybe

- **Backlinks tool** — agent tools for `get_backlinks(file)` and `get_outgoing_links(file)`. Independently useful but not blocking any phase above. The agent can grep for wikilinks today.
