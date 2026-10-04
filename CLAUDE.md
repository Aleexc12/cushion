# Cushion

Local-first markdown workspace inspired by Obsidian, with integrated AI chat. Bun monorepo — run with `bun install && bun run dev`.

## Stack

- **Frontend** (`apps/frontend`): Vite, React 19, Tailwind 3, CodeMirror 6, Zustand 5. Communicates with the coordinator via Electron IPC. Runs standalone in the browser or as Electron's renderer.
- **Electron** (`apps/electron`): Electron 35 shell with an integrated coordinator (`src/main/coordinator/`) that handles file CRUD, workspace state, config management, file watching, dictation (Sherpa ONNX), and AI chat (OpenCode server) via IPC — no separate server process.
- **Shared packages**: `packages/types` for shared TypeScript types, `packages/tsconfig` for shared configs.

## Key Concepts

- **File-first**: title = filename; editing the title renames the file.
- **WYSIWYG markdown**: CodeMirror with custom extensions that hide syntax when the cursor is off-line (`lib/codemirror-wysiwyg/`).
- **Wiki-links**: `[[note]]`, `[[note#header]]`, `[[note|display]]` with fuzzy resolution and backlinks.
- **AI chat sidebar**: OpenCode SDK (`@opencode-ai/sdk`) powers the chat panel (`components/chat/`). The Electron coordinator spawns the OpenCode server; the frontend connects via the SDK client (`lib/opencode-client.ts`).
- **Dictation**: Local speech-to-text via Sherpa ONNX. Coordinator manages model downloads and the Sherpa process; frontend has dictation button, settings, and post-processing UI.
- **Rich content**: Excalidraw drawings, native PDF viewer (pdfjs-dist), KaTeX math, code blocks with syntax highlighting (highlight.js).
- **Singleton clients**: single IPC/SDK instances shared across the app.

## Frontend Layout

- `src/Home.tsx` — root component, layout shell
- `components/chat/` — AI chat sidebar (sessions, messages, tools, model/agent selectors)
- `components/editor/` — editor panel, tabs, PDF viewer, Excalidraw, dictation button
- `components/workspace/` — file tree, file browser, trash viewer, workspace switcher
- `components/settings/` — appearance, dictation, shortcuts, files, editor settings
- `components/ui/` — shared UI primitives
- `stores/` — Zustand stores (chat, workspace, dictation, appearance, explorer, shortcuts)
- `lib/codemirror-wysiwyg/` — all CodeMirror extensions (hide-markup, widgets, wiki-links, tables, focus-mode, slash-commands, AI diff)
- `lib/shortcuts/` — keyboard shortcut system
- `hooks/` — React hooks (prompt editor, PDF, dictation, etc.)

## Coordinator (`apps/electron/src/main/coordinator/`)

Handles all backend concerns via IPC:
- `workspace-manager.ts` / `workspace-watcher.ts` — file CRUD, file watching (chokidar)
- `config-manager.ts` / `config-watcher.ts` — user config persistence
- `opencode-server.ts` / `opencode-config.ts` — spawns and manages the OpenCode AI server
- `sherpa-manager.ts` / `sherpa-binary-manager.ts` / `sherpa-model-manager.ts` — local dictation engine
- `hotkey-manager.ts` — global hotkeys
- `trash-manager.ts` — soft-delete / restore
- `ipc-router.ts` — IPC handler registration
- `handlers/` — individual IPC handler modules

## Design Principles

Clean, modular code. Small focused modules, clear separation of concerns, no unnecessary abstractions.

## Scripts

- `bun run dev` — frontend only (Vite dev server)
- `bun run dev:electron` — full Electron app with coordinator
- `bun run build` / `bun run build:electron` — production builds
- `bun run lint` — type-check + color guardrails + pdfjs asset verification
- `bun run test` — vitest across all packages

## Testing

- Use `bun run test` (not `bun test`) from the root. Bare `bun test` uses Bun's native runner which skips vitest's jsdom environment, causing false failures in frontend tests.
- Frontend tests: `vitest run` with jsdom environment (`apps/frontend/vitest.config.ts`).

## INSTRUCTIONS

- Do not start or end my server, I will have it on hot reload.
- `/inspo` folder has apps that inspire cushion. Ignore this folder if not asked to search information here.
- Use `ls` if you can't find any folder.
- Use globals.css colors, do not add new colors. If you really need to, ask first.
- `docs/implemented/` has architecture docs (ARCHITECTURE.md, tables-api.md, diff.md) — consult when working on related features.

## Agent skills

### Issue tracker

Issues live in GitHub Issues for `Aleexc12/cushion`, managed with the `gh` CLI. See `docs/agents/issue-tracker.md`.

### Triage labels

Default vocabulary: `needs-triage`, `needs-info`, `ready-for-agent`, `ready-for-human`, `wontfix`. See `docs/agents/triage-labels.md`.

### Domain docs

Single-context: one `CONTEXT.md` and `docs/adr/` at the repo root. See `docs/agents/domain.md`.
