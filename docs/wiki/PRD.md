# Cushion Wiki — PRD

High-level product spec for the LLM-maintained wiki feature and everything required to ship it. Covers the full scope — versioning/sequencing is a separate exercise.

## Goal

Turn Cushion into a workspace where an LLM incrementally builds and maintains a persistent markdown wiki from the user's source material. User curates sources and asks questions; the LLM does the bookkeeping. Differentiates Cushion from "Obsidian plus an AI sidebar" by shipping the wiki maintenance loop as a first-class capability — no CLAUDE.md, no CLI, no setup.

See `docs/old_karpathy/Wiki.md` for design rationale and `docs/wiki/llm_wiki.md` for the underlying pattern. Many concepts below are adapted from [graphify](https://github.com/safishamsi/graphify) (vendored at `inspo/graphify/`) — specific source files are cited inline as `(ref: ...)` so SPEC work can pull patterns directly.

## Scope

**In scope**
- Ingest, query, and lint operations over a `sources/` + `wiki/` workspace layout.
- URL fetching for web articles, PDFs, and video/audio (reusing Sherpa).
- Staleness and confidence tracking surfaced in the UI.
- A global tag system usable by the wiki and by ordinary notes.

**Out of scope**
- Graph visualization and community detection (revisit once the wiki workflow has real usage).
- Canvas-style spatial views (Excalidraw covers this need).
- MCP server / external CLI.
- Code-aware extraction (no AST, no tree-sitter). Wiki processes documents, not source trees.

## Prerequisites

Features that must exist before the wiki workflow is buildable. Each is independently useful; none exists purely for the wiki.

### P1. Tag system (app-wide)

Markdown files get tags via YAML frontmatter (`tags: [a, b]`) and inline `#tags` in the body. Tags are indexed workspace-wide, shown in a sidebar pane (list of tags + counts, click to filter the file tree). Reference implementations: Zettlr, Tangent.

- Frontmatter and inline tags both count.
- Tag index maintained incrementally on file-watcher events.
- Namespaced tags (`#wiki/stale`, `#source/pdf`) render as hierarchy.
- Click a tag → file tree filters to matching files.

### P2. Editor status bar

Wiki pages carry meaningful metadata (`sources`, `updated`, `confidence`) that must be visible, not buried in raw YAML. Without this, the rot-protection story is invisible. Solved with a sticky bottom-right status overlay on every markdown file (Obsidian-style) — not an inline editor transform.

- Sticky bar flush to the editor's bottom-right corner (absolute, `bottom-0 right-0`, top+left border, `rounded-tl-md`, background matches `--md-bg`). Applies to all markdown files, not just wiki pages.
- Baseline chips (always): `N words`, `N chars`, `N tags`, whitespace-separated (no dots). Tags count pulled from the existing tag store (frontmatter + inline).
- Wiki-specific chips (conditional on frontmatter presence): `N sources`, `updated Xd ago`, confidence label (`EXTRACTED` / `INFERRED` / `AMBIGUOUS`).
- Display-only at ship. Click-through actions (source chip → open source file; stale chip → run lint on this page) wire in alongside W1/W3.
- **Not doing**: hiding the YAML block in the editor, or registering a frontmatter parser extension on `@lezer/markdown`. Consequence: `---\n...\n---` at the top of a wiki page renders as two horizontal rules with raw YAML between them. Status bar still parses it correctly (regex-based, independent of CodeMirror). Dormant `FrontMatter` node handlers already exist in `hide-markup.ts` / `inline-replace.ts` / `list-commands.ts` — wiring a parser extension later is ~10 lines if the two-HR look becomes annoying.
- **Deferred**: `stale` chip requires the W1 ingest manifest (source-hash tracking) to compute. Add the chip when W1 lands.

### P3. Source folder convention

A workspace can designate subfolders as `sources/` (immutable inputs) and `wiki/` (LLM-owned outputs). File tree visually distinguishes them (icon / section header). Users can rename these in workspace settings but defaults match graphify's convention.

- Workspace config stores `{ sourcesDir, wikiDir }`.
- Writes to `sources/` from the LLM are blocked at the coordinator layer (defense in depth).
- `wiki/` marked as LLM-managed so users are nudged not to hand-edit (warning banner, not hard block).
- `.cushionignore` at workspace root — gitignore-syntax patterns to exclude folders/files from ingest (e.g. `archive/`, `*.draft.md`). Applies to ingest/lint, not to the editor. (ref: `inspo/graphify/graphify/detect.py` for ignore-file parsing; `.graphifyignore` in graphify README)

### P4. URL ingest pipeline

A "Add from URL" dialog accepts a URL and routes to the correct fetcher:
- **Web articles** → readability-style extraction → markdown saved to `sources/`. Replaces Obsidian Web Clipper.
- **PDFs** → download → saved as-is (existing PDF viewer handles rendering).
- **Videos/audio (YouTube, direct links)** → audio extraction + local transcription via existing Sherpa pipeline → transcript markdown to `sources/`.
- **Images** → download to `sources/assets/`.

Metadata (author, captured_at, source_url) written as frontmatter on the saved file — propagates through to wiki pages on ingest.

Reference implementation: `inspo/graphify/graphify/ingest.py` — has working fetchers for tweet (oEmbed), arXiv (abstract scrape), generic webpage (html2text), PDF and image binary download, and YouTube (delegates to local audio transcription). `_detect_url_type()` dispatches by URL pattern. `safe_fetch()` / `validate_url()` in `inspo/graphify/graphify/security.py` handle size caps, timeouts, and blocking `file://` redirects.

### P5. OpenCode agent configuration

Ingest/lint are long-running agent tasks, not chat turns. Needs:
- A dedicated "wiki maintainer" agent with its own system prompt, tool whitelist (read/write in `wiki/`, read-only in `sources/`), and model preference.
- Progress surfacing — the user sees which source is being processed, which pages are being updated.
- Cancellation.

Investigation needed: does OpenCode SDK already support multiple named agents with distinct permissions, or does this require coordinator-side plumbing?

### P6. Log / timeline view

A dedicated panel renders `wiki/log.md` as a chronological feed of ingest/query/lint events. Simpler than parsing arbitrary markdown — entries follow a fixed format (`## [YYYY-MM-DD HH:MM] <op> | <title>`) and are rendered as timeline cards with expand-for-detail.

- Doubles as a "what changed" surface for trust.
- "Revert this ingest" action per entry (uses git under the hood; surfaces a diff).

## The wiki feature itself

Once P1–P6 are in place, the wiki layer is a relatively thin addition.

### W1. Ingest

Button in workspace toolbar: **Ingest sources**. Runs the wiki agent over new/changed files in `sources/`.

- Incremental by default — manifest tracks source hashes, skips unchanged.
- Frontmatter-aware hashing (metadata changes to wiki pages don't invalidate cache; source content hash is separate).
- Each source → LLM reads, extracts concepts/entities, creates or updates pages in `wiki/`, updates `wiki/index.md`, appends entry to `wiki/log.md`.
- Every claim on a wiki page is tagged `EXTRACTED` / `INFERRED` / `AMBIGUOUS` in frontmatter. No default; LLM must choose. (ref: `inspo/graphify/ARCHITECTURE.md` "Confidence labels" table; schema enforcement in `inspo/graphify/graphify/validate.py`)
- Page frontmatter includes `sources: [hash1, hash2, ...]` for staleness detection. (ref: cache + manifest pattern in `inspo/graphify/graphify/cache.py` and `inspo/graphify/graphify/detect.py` — SHA256 over source content, re-runs skip unchanged)
- Contradictions detected during ingest: newer source wins on the page, old+new claim+sources logged to `log.md`.

### W2. Query

Already exists — chat sidebar. The wiki agent reads `wiki/index.md` first, then drills into relevant pages, answers with `[[wikilink]]` citations.

- New: "Save to wiki" action on assistant messages. Opens a dialog to pick a page (new or existing) and writes the answer. (ref: `save_query_result()` in `inspo/graphify/graphify/ingest.py` — frontmatter shape for filed answers, including `question`, `date`, `source_nodes`)
- No automatic filing of query responses — explicit user action only.

### W3. Lint

Button: **Check wiki health**. Runs structural checks locally (no LLM) plus an optional LLM pass for contradictions.

Structural checks (ref: "Knowledge Gaps" section of `inspo/graphify/graphify/report.py` — isolated-node + thin-community + high-ambiguity heuristics map directly to orphan / high-ambiguity page checks):
- Orphan pages (no incoming `[[wikilinks]]`)
- Stale pages (source hash changed since last ingest)
- High-ambiguity pages (>20% claims marked `AMBIGUOUS`) — threshold lifted from graphify's report
- Broken wikilinks
- Source files present but never ingested

LLM pass (opt-in, costs tokens):
- Cross-page contradictions on the same entity.
- Suggested missing pages based on recurring concepts.
- **Surprising connections** — pairs of wiki pages that share sources or concepts but aren't linked yet. Agent proposes `[[wikilinks]]` to add *between wiki pages* (never edits `sources/`). User approves or dismisses per-issue. (ref: "Surprising Connections" section of `inspo/graphify/graphify/report.py`; ranking logic in `inspo/graphify/graphify/analyze.py`)
- **Suggested questions** — 4–5 questions the wiki is uniquely positioned to answer, surfaced as chat-prompt buttons. Good re-entry point when opening a wiki cold. (ref: `suggested_questions` block at the end of `generate()` in `inspo/graphify/graphify/report.py`, produced by `inspo/graphify/graphify/analyze.py`)

Results render in a panel with per-issue "fix" or "dismiss" actions. Lint runs append to `log.md`.

### W4. Wiki page format

```yaml
---
title: Entity Name
created: 2026-04-14T10:00:00Z
updated: 2026-04-20T14:30:00Z
sources:
  - hash: abc123
    path: sources/paper.pdf
    captured_at: 2026-04-14
confidence: EXTRACTED   # dominant confidence of claims on page
tags: [topic-a, topic-b]
---
```

Body: markdown with `[[wikilinks]]` and inline confidence badges on synthesized claims. Cross-references are natural wikilinks — Cushion already resolves them. (ref: graphify's generated page shape in `inspo/graphify/graphify/wiki.py` — `_community_article()` and `_god_node_article()` show a workable layout for "key claims / relationships / sources / audit trail" sections; community/god-node framing itself is out of scope but the per-page anatomy transfers)

### W5. `wiki/index.md` and `wiki/log.md`

- `index.md` — flat catalog, LLM-maintained. Sections by tag or category. Each entry: `[[page]] — one-line summary`. Read first on every query. (ref: `_index_md()` in `inspo/graphify/graphify/wiki.py`)
- `log.md` — append-only, fixed-prefix format. Rendered by P6 timeline view.

## Risks and open questions

1. **Epistemic drift / rot.** Partially mitigated by source-hash staleness and explicit confidence tagging. Lint surfaces known rot patterns but can't catch confidently-wrong pages. This is a known unsolved problem; the UI mitigations are the differentiator, not a cure.
2. **OpenCode agent isolation.** Unclear if the SDK supports per-agent tool permissions today. If not, coordinator must gate tool calls by agent ID — non-trivial.
3. **URL fetching and extraction quality.** Readability extraction is a solved problem (Mozilla Readability) but rendering JS-heavy pages may require a headless browser. Scope decision: plain-fetch first, headless later if needed.
4. **Tag system performance.** Incremental indexing on every file-watcher event must be cheap. Target: <50ms index update for single-file changes at 1000-file scale.
5. **What happens when the user edits a wiki page by hand?** Probably: detect by content hash vs last-ingest hash; show "manually edited" badge; next ingest that touches this page asks before overwriting.

## Non-goals (explicit)

- Not a research tool. No RAG, no vector store, no embeddings. Flat index + LLM reasoning.
- Not code-aware. Cushion is a document workspace. `sources/` accepts text, PDFs, images, video/audio.
- Not a team collaboration product. Single-user, local-first, git-versioned.
