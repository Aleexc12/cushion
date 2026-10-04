# Phase 1: Ingest Pipeline — PRD

## Goal

User drops files into `sources/`, clicks a button, and gets a structured knowledge base in `knowledge/`. No setup, no schema writing, no CLI.

## User flow

1. User creates a `sources/` folder in their vault and drops files into it (any type: md, pdf, txt, images, etc.)
2. File watcher detects new or changed files in `sources/` — a dirty dot appears on the `sources/` folder in the file tree.
3. An "Ingest" button appears in the chat bar top area.
4. User clicks "Ingest."
5. A background OpenCode session spawns and processes source files.
6. On first run, `knowledge/` and its subfolders (`concepts/`, `connections/`, `qa/`) plus `index.md` and `log.md` are created automatically.
7. When done, a dialog appears: "Ingestion complete. Created X articles, updated Y."
8. Dirty dot clears.

## Ingest behavior

- **Incremental by default.** Tracks SHA-256 hashes of source files. Only processes new files and files changed since last ingest.
- **First run** processes everything in `sources/`.
- **Contradictions** — newer source wins, page gets silently updated.
- **One background OpenCode session** orchestrates the ingest. It can spawn subagents per source file so one failure doesn't block the rest.
- **Compile prompt** is hardcoded in the app — tells the LLM the article format, frontmatter schema, wikilink conventions, and how to maintain `index.md`.

## What gets produced

- `knowledge/concepts/*.md` — one article per atomic topic
- `knowledge/connections/*.md` — cross-cutting insights linking 2+ concepts
- `knowledge/index.md` — table of every article with a one-line summary (the retrieval mechanism)
- `knowledge/log.md` — append-only record of each ingest run
- State file (JSON, gitignored) tracking source file hashes and ingest timestamps

## Out of scope for Phase 1

- Query integration (Phase 2)
- Lint (Phase 3)
- Provenance UI / freshness badges (Phase 4)
- Backlinks agent tool (future/maybe)
- Customizable schema file
- Auto-ingest / ambient hooks
