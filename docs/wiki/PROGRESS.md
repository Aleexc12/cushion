# Cushion Wiki — Progress

Companion to `PRD.md`. Tracks what has shipped vs. what remains. Keep entries terse — one line per fact.

## Status legend
`[x]` done · `[~]` partial · `[ ]` not started

## Prerequisites

### P1. Tag system — `[x]`
Shipped end-to-end. Deviations from the original plan noted below.

- Coordinator: `apps/electron/src/main/coordinator/tag-indexer.ts` — `TagIndexer` owns `file→tags` + `tag→files` maps, 50ms debounced flush.
- Extraction: frontmatter `tags:` (string or array, via `yaml`) + inline `#tag` / `#parent/child`. Scrubs fenced/inline code. Regex requires `[\p{L}_]` start → no `#123`, no `## heading` collisions.
- Prefix-inclusive lookup: `filesForTag('wiki')` returns files tagged `wiki`, `wiki/stale`, etc.
- IPC: `handlers/tags.ts` + `coordinator:tags/updated` broadcast; wired in `ipc-router.ts`, re-inits on workspace open, clears on close, subscribes to `setOnFilesChanged`.
- Types: `TagInfo`, `TagNode`, `TagUpdateEvent` in `packages/types/src/index.ts` (+ RPC entries).
- Frontend store: `stores/tagStore.ts` (Zustand + `subscribeWithSelector`); derives tree via `lib/tag-tree.ts` (pure, tested in `lib/tag-tree.test.ts`, 7 cases). Client methods: `listTags`, `filesForTag`, `onTagsUpdated` in `lib/coordinator-client.ts`.
- Sidebar: Files/Tags toggle lives in the **bottom bar** (Tag icon) — not the top tab switcher from the plan. `TagsPane.tsx` + `TagTreeRow.tsx` reuse file-tree row grammar (chevron + label + count).
- Virtual tab: `tag://<path>` scheme. `TagResultsView.tsx` collects self + descendant files, opens with `setPendingFind(filePath, '#${tag}')`. Branches in `editor-path.ts`, `EditorPane.tsx`, `PaneTabBar.tsx`, `useConfigSync.ts`.
- Editor decoration: `lib/codemirror-wysiwyg/tag-decoration.ts` — `StateField` emits `.cm-tag-hash` / `.cm-tag-segment` (with `data-tag-path` accumulator) / `.cm-tag-sep`. Skips `FencedCode`/`InlineCode`/`URL`/`Link`/`Image`/`Comment` via syntaxTree. Plain click (not Ctrl) navigates via injected callback set by `CodeEditor.tsx`. Pill styling in `styles/markdown-editor.css:1094-1131`.

Deferred (matches plan): autocomplete on `#`, bulk rename, tag colors, persisted index, `.cushionignore`.

### P2. Editor status bar — `[x]`
Shipped. `components/editor/EditorStatusBar.tsx` — sticky bottom-right bar flush to the editor corner (top+left border, `rounded-tl-md`, `--md-bg` background, `text-muted-foreground`, whitespace-separated). Mounted in `EditorPane.tsx` beside `RecordingOverlay`, hidden on tag tabs and non-markdown. Reads cached `fileState.frontmatter` (re-parsed in `workspaceStore.updateFileContent`) and `tagStore` for tag count. Body-only word/char count (excludes frontmatter bytes). Display-only — click-through on source/stale chips wires with W1/W3.

Deferred: `stale` chip (needs W1 manifest); confidence color-coding; YAML-hiding in editor (explicitly not doing).
### P3. Source folder convention — `[x]`
Opt-in, hardcoded `sources/` + `wiki/`. No `.cushionignore`, no coordinator write-guard (deferred to P5 — OpenCode bypasses coordinator).

- Constants: `SOURCES_DIR`/`WIKI_DIR` in `coordinator/constants.ts`; frontend helpers in `lib/wiki-paths.ts` (`isSourcesPath`, `isWikiPath`, `specialFolder`).
- Config: `CushionSettings.wikiEnabled` (false) + `warnBeforeEditingWiki` (true) in `packages/types/src/config.ts` + `lib/config-defaults.ts`. Persisted via existing `useConfigSync` → `.cushion/settings.json`.
- Enable action: `FilesSettings.tsx` "Wiki" section — button calls `client.createFolder('sources')` then flips `wikiEnabled`. `wiki/` stays lazy (W1).
- Tint: `FileTreeRow.tsx` applies `text-[var(--accent-primary)]` to `FolderIcon` only at `depth === 0` and only when `wikiEnabled`. Hover-revealed chevron inherits default color (cosmetic).
- Soft lock: `CodeEditor.tsx` gains `readOnly` prop via `Compartment` wrapping `EditorView.editable`. `EditorPane.tsx` computes `isLocked = wikiEnabled && warnBeforeEditingWiki && isWikiPath(fp) && !unlockedTabs.has(tabId)`; renders `WikiLockBanner` with "Edit anyway" → `unlockTab`.
- Store: `workspaceStore` adds `unlockedTabs: Set<string>` + `unlockTab`, cleaned in `removeTab`/`closePane`. Ephemeral — not in `partialize`.

Deferred: `.cushion/ignore` (revisit at W1); rename support for dir names; OpenCode-side `sources/` write restriction (P5).
### P4. URL ingest pipeline — `[~]`
v1 shipped: web / PDF / image. YouTube + audio deferred to v2.

- Coordinator module: `coordinator/url-ingest/` — `security.ts` (SSRF: scheme allowlist, dotted-quad + IPv6 CIDR checks, DNS re-resolution, metadata-host block, re-validated on every redirect hop), `fetch.ts` (streaming `safeFetch` with 50MB cap, 15s `AbortSignal.timeout`, manual redirect loop capped at 5 with body cancel), `detect.ts` (`resolveType` — Content-Type authoritative, URL ext fallback; `imageExtension` mime→ext; `safeFilename`), `frontmatter.ts` (W4-shape YAML via `yaml`), `fetchers/webpage.ts` (`extractArticle`: linkedom + Readability + Turndown), `fetchers/binary.ts` (`saveBytes` — ensures parent dir, routes via `saveFileBase64` so watcher fires), `index.ts` (one fetch, dispatch by resolved type, 10MB cap enforced post-fetch on HTML, 1000-attempt `_N` collision loop).
- IPC: `handlers/url-ingest.ts` — discriminated union result (`ok` | `INVALID_URL`/`BLOCKED`/`FETCH_FAILED`/`TOO_LARGE`/`EXTRACTION_FAILED`); registered in `ipc-router.ts`.
- Types: `UrlIngestType`, `UrlIngestErrorCode` in `packages/types/src/index.ts`; `workspace/ingest-url` RPC in `rpc.ts`.
- Frontend: `coordinator-client.ts#ingestUrl`; `components/workspace/AddFromUrlDialog.tsx` (idle/loading/error states, Retry); `FileTreeItemActions.tsx` gates "Add from URL…" on `node.path === SOURCES_DIR`; `FileBrowser.tsx` owns dialog state + splits binary vs markdown open path.
- Deps (coordinator): `linkedom`, `@mozilla/readability`, `turndown`.
- Tests: `security.test.ts`, `detect.test.ts`, `fetch.test.ts`.

Deferred: binary metadata sidecar (PDFs/images have no frontmatter — decision pending: per-file `.meta.yml` vs central manifest vs let W1 infer); YouTube/audio (v2, needs yt-dlp); arxiv/tweet specializers; headless browser for JS-heavy sites; settings surface (caps hardcoded).

### P5. OpenCode wiki-maintainer agent — `[ ]`
### P6. Log / timeline view — `[ ]`

## Wiki feature (W1–W5)
All `[ ]`. Blocked on P2–P6.

## Open decisions carried forward
- Segment click is plain click, not Ctrl-click — revisit if it conflicts with future text-selection gestures in the pill.
- `.cushionignore` not yet honored by the tag indexer (indexes all `.md`). When P3 lands, plumb the ignore matcher into `TagIndexer.updateFile`/`init`.
- Frontmatter util (`apps/frontend/lib/frontmatter.ts`) was **not** hoisted to shared; coordinator re-implements minimal YAML parsing in `tag-indexer.ts`. If P2 needs the same logic on both sides, hoist to `packages/shared` then.
