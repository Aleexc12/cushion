# Cushion: notes on a wiki-maintenance feature

## The problem worth solving

Karpathy's LLM Wiki gist diagnoses something real: the bottleneck in personal knowledge bases is bookkeeping (cross-references, summaries, contradiction tracking), and LLMs make bookkeeping nearly free. RAG re-derives understanding on every query and nothing accumulates. A persistent, LLM-maintained wiki layer between raw sources and the user is the fix.

This is a different problem from "search my files faster." Search is cheap; compounded synthesis is what's missing.

## Why this fits Cushion specifically

Cushion already has the pieces nobody else has assembled: an agent in the sidebar with file access, OpenCode under the hood, markdown vault, editor signals about user activity. The friction in Karpathy's pattern (install Obsidian, configure Claude Code, write a CLAUDE.md schema, learn the workflow) is exactly what an integrated app removes. Shipping wiki maintenance as a first-class capability differentiates Cushion from "Obsidian plus an AI sidebar."

## Prior art

**Karpathy's gist** describes a three-layer architecture: raw sources (immutable), wiki layer (LLM-owned), and a schema file (CLAUDE.md) defining conventions. Three operations: ingest (source -> wiki pages), query (search index -> synthesize answer), lint (find contradictions, staleness, orphans). The LLM reads a structured `index.md` instead of using RAG — at personal scale (50-500 articles), the model reasoning over summaries outperforms vector similarity. Obsidian is used as the viewer, but no Obsidian-specific features are required. Everything is plain markdown with wikilinks.

**Cole Medin's claude-memory-compiler** is a concrete implementation of the pattern, but the raw data is Claude Code conversations instead of web articles. It adds: Claude Code hooks for automatic capture (SessionStart, SessionEnd, PreCompact), a two-phase extraction (cheap flush -> expensive compile), SHA-256 state tracking for incremental processing, and end-of-day batched compilation. Article types: concepts (atomic knowledge), connections (cross-cutting insights), Q&A (filed query answers). Uses the Claude Agent SDK to run compilation and query as background processes.

Neither project depends on any Obsidian feature. Obsidian is just a markdown viewer with a nice graph. The actual machinery is: give an LLM access to files and good instructions. Cushion already has that via OpenCode.

## What Cushion takes from each

From Karpathy: the index-guided retrieval strategy (no RAG), the three operations (ingest, query, lint), the emphasis on summaries as the retrieval mechanism, and the insight that maintenance is the bottleneck humans abandon.

From Cole Medin: the article formats (frontmatter with sources, created, updated), the compile prompt structure (feed schema + index + existing articles + new source, let the LLM write files directly), the incremental state tracking (SHA-256 hashes, skip unchanged), the lint checks (7 structural + semantic health checks), and the query prompt pattern (read index first, pick relevant articles, synthesize with citations).

## What Cushion does differently

**Manual ingest, not ambient hooks.** Karpathy's pattern requires discipline (remember to run CLI commands). Cole Medin automates via hooks (fires on every session end). Cushion uses a button — explicit enough that the user knows tokens are being spent, but low-friction enough that they'll actually do it. Ingest is incremental by default (only new/changed source files).

**Source folder, not conversations.** Cole Medin compiles knowledge from Claude Code transcripts. Cushion's `sources/` folder accepts anything the user drops in: PDFs, markdown, text, images. OpenCode handles text extraction. The compile prompts don't care where the text came from.

**No setup.** The folder structure (`sources/`, `knowledge/`, `knowledge/index.md`), article formats, and compile instructions are all built into the app. The user never writes a CLAUDE.md schema or configures hooks.

**Query through the chat sidebar.** No separate CLI tool — the agent reads `index.md` when the user asks a question and synthesizes answers with wikilink citations. Filed-back Q&A articles make the knowledge base smarter over time.

## The rot problem

The unsolved problem in Karpathy's pattern is epistemic drift: every ingest is a lossy compression with an opinionated synthesizer. Errors compound across rounds. Three months in, the wiki disagrees with the sources in ways no one notices. Lint catches contradictions the model can see, not confidently wrong pages no later source happens to challenge.

Cushion's answer: each knowledge article carries metadata recording which source files contributed to it and when (the `sources:` frontmatter field). Staleness is detectable — if a source file changes after compilation, the article is stale. This can be surfaced in the UI (provenance hover, freshness badges) rather than trusted blindly. This is a product surface, not just a feature.

## Things to push back on

- Don't ship wiki maintenance as a one-shot bootstrap skill. The value is in the maintenance loop on every subsequent ingest, not the initial generation.
- Don't build a `traverse` tool as a separately-named capability. It over-specifies the search strategy. Backlinks plus a competent agent is enough.
- Don't lead with token-burning maintenance. Ingest should be manual and incremental so the user controls cost.

## Open design question (resolved)

When the agent finds a contradiction during ingest, it silently updates the page. The newer source wins. The user can review changes via git diff or the file tree — the knowledge base is just markdown files. This keeps the ingest flow simple: click button, wait, done.
