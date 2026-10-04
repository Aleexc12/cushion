# Phase 1 Ingest — Implementation Plan

## Approach

Single visible session in the chat sidebar. User clicks "Ingest," a new session is created, and the compile prompt is sent. The agent works visibly — the user sees it reading sources and writing articles. No background infrastructure needed.

## Architecture Decision

**Single prompt, agent orchestrates.** Send one `session.prompt()` with the compile instructions as the `system` param and a short user-visible message. The agent reads `sources/`, decides what to do, writes `knowledge/` files. It can spawn subagents on its own if needed.

Alternatives considered:
- **`SubtaskPartInput` per file** — SDK supports subtask parts that fan out to separate agent calls. More like Cole Medin's per-file approach. Good for resilience but adds complexity. Can upgrade to this later.
- **Multiple `promptAsync()` calls** — Fire-and-forget per file. Unclear if sessions handle concurrent prompts. Not worth the risk for v1.
- **Background session** — Invisible to user, reports progress via toasts. Requires new infrastructure for session orchestration, progress tracking, completion detection. Deferred.

## Key SDK Features

```ts
client.session.prompt({
  sessionID: string,
  system?: string,          // custom system prompt (schema + conventions)
  parts: [TextPartInput],   // the visible user message
  tools?: Record<string, boolean>,  // enable/disable specific tools
  noReply?: boolean,        // send without triggering AI response
  agent?: string,
  model?: { providerID, modelID },
});
```

`SubtaskPartInput` for future per-file fan-out:
```ts
{
  type: "subtask",
  prompt: string,
  description: string,
  agent: string,
  model?: { providerID, modelID },
}
```

## What Exists (Reuse)

| Need | How |
|------|-----|
| Send prompt programmatically | `sendPrompt()` from `chatStore` — handles session creation, optimistic messages, error recovery |
| Force new session | `client.session.create({ directory })` then set as active |
| Agent reads/writes files | OpenCode already has file access in the vault |
| Detect `sources/` changes | `client.onFilesChanged()` → filter for `sources/` paths |
| Check if `sources/` exists | `client.listFiles('sources')` |
| Create `knowledge/` folders | `client.createFolder()` |
| Persist ingest state | `client.writeConfig()` / `client.readConfig()` → `.cushion/ingest-state.json` |
| Toast notifications | `useToast()` with loading/success/error variants |
| File tree refresh | Automatic — workspace watcher picks up new files |
| Completion detection | `chatStore.sessionStatus` tracks busy → idle transitions |

## What Needs Building

### 1. Compile Prompt

Adapted from Cole Medin's `compile.py`. Two parts:

**System prompt** (via `system` param — invisible to user):
- Full schema: article format, frontmatter spec, wikilink conventions
- Folder structure: `sources/` → `knowledge/{concepts,connections,qa}/`
- Rules: encyclopedia style, cross-references, index maintenance
- Incremental behavior: list of files to process (new/changed only)

**User-visible message** (via `parts`):
- Short: "Ingesting 5 new files from sources/"
- Or just: "/ingest"

Key difference from Cole Medin: we don't pre-read all existing articles into the prompt. The agent has file access and can read them itself based on the index. This keeps the prompt smaller.

### 2. Ingest Trigger (Frontend)

**Option A: Button above PromptInput** — Like `TodoDock` or `QuestionDock`, a dock element that appears when `sources/` has unprocessed files.

**Option B: Slash command** — User types `/ingest`, local command handler intercepts it and runs the ingest flow.

**Option C: Both** — Button for discoverability, slash command for power users.

The trigger needs to:
1. Read ingest state from `.cushion/ingest-state.json`
2. List files in `sources/`
3. Diff against stored hashes to find new/changed files
4. Build the compile prompt with the file list
5. Force-create a new session (`client.session.create()`)
6. Set it as active session in `chatStore`
7. Send the compile prompt

### 3. Dirty Dot on `sources/`

- Listen to `onFilesChanged`, filter for `sources/` paths
- Track dirty state in `explorerStore` (new `dirtyPaths: Set<string>`)
- Render a small dot in `FileTreeRow` when path is in `dirtyPaths`
- Clear after successful ingest

### 4. Incremental State Tracking

`.cushion/ingest-state.json`:
```json
{
  "ingested": {
    "sources/file.pdf": {
      "hash": "sha256...",
      "ingestedAt": "2026-04-07T14:30:00Z"
    }
  },
  "lastIngestTime": "2026-04-07T14:30:00Z",
  "ingestCount": 3
}
```

SHA-256 hashing needs to happen somewhere:
- **Option A: Frontend** — Read file as base64 via `readFileBase64()`, hash with Web Crypto API
- **Option B: Coordinator** — New IPC handler that hashes files server-side (cleaner, handles large files)
- **Option C: Skip for v1** — Process all files every time, add incremental later

### 5. Session Flow

```
User clicks Ingest
  → Read .cushion/ingest-state.json
  → List sources/ files
  → Diff against stored hashes (or skip for v1)
  → Build compile prompt
  → client.session.create({ directory })
  → Set new session as activeSessionId
  → client.session.prompt({ sessionID, system: SCHEMA, parts: [{ type: 'text', text: 'Ingesting N files...' }] })
  → Session appears in chat sidebar, agent works visibly
  → Watch sessionStatus for busy → idle transition
  → On idle: update .cushion/ingest-state.json, clear dirty dot, show toast
```

### 6. Completion

When `sessionStatus` transitions from `busy` to `idle`:
- Update ingest state file with new hashes
- Clear dirty dot on `sources/`
- Show toast: "Ingestion complete"
- For "Created X, updated Y" counts: parse the agent's log.md entry, or just skip counts in v1

## File Changes

| File | Change |
|------|--------|
| `stores/chatStore.ts` | New `sendIngestPrompt()` action |
| `stores/explorerStore.ts` | Add `dirtyPaths` state + `markDirty`/`clearDirty` |
| `components/chat/ChatSidebar.tsx` | Add `IngestDock` or `IngestButton` |
| `components/workspace/FileTreeRow.tsx` | Render dirty dot |
| `lib/ingest-prompt.ts` (new) | Compile prompt builder |
| `lib/ingest-state.ts` (new) | Read/write `.cushion/ingest-state.json` |
| `src/Home.tsx` | Wire up `onFilesChanged` listener for `sources/` |

## Build Order

1. **Compile prompt** (`lib/ingest-prompt.ts`) — the schema + prompt builder
2. **Ingest action** (`stores/chatStore.ts`) — force-create session + send prompt
3. **Ingest button** (`components/chat/`) — trigger in the UI
4. **Dirty dot** (`explorerStore` + `FileTreeRow`) — visual indicator
5. **Incremental state** (`lib/ingest-state.ts`) — SHA-256 tracking
6. **Completion handling** — status detection, state update, toast

## Reference: Cole Medin's Compile Prompt Structure

From `scripts/compile.py`, the prompt includes:
1. Role: "You are a knowledge compiler"
2. Full AGENTS.md schema (article formats, frontmatter, conventions)
3. Current wiki index (`knowledge/index.md`)
4. All existing articles (pre-read and concatenated)
5. The source file content
6. Explicit rules (extract 3-7 concepts, create articles, update index, append log)
7. File paths to write to
8. Quality standards (frontmatter required, 2+ wikilinks, 3-5 bullet points, etc.)

For Cushion: items 1-2 go in `system` param. Items 3-4 are omitted (agent reads them itself). Items 5-8 go in the user-visible prompt along with the file list.
