# Extensions

CodeMirror edits markdown and Monaco edits other text files.
Extensions are viewers and editors for the file types neither of them handles.
The API gives an extension its own render pane and nothing else.
It has no hooks into CodeMirror, the sidebar, the file tree, or any other global UI.
Extension code still runs in the app's renderer, so nothing but convention keeps it out of the rest of the DOM.

The plugin system in [#48](https://github.com/Aleexc12/cushion/issues/48) replaces this design.
This doc describes the code as it stands until then.

## Extension types

### Bundled

Bundled extensions ship with the app and load through Vite imports.
Their manifests live in `packages/extension-api/manifests/`.
Users can disable them in settings.

- Image Viewer: png, jpg, jpeg, gif, svg, webp, bmp, ico (read-only, binary)
- PDF Viewer: pdf (binary, chunked reads, annotations)
- Excalidraw: excalidraw (read-write, JSON)

### External

Users install external extensions into `~/.cushion/extensions/`.
The coordinator discovers them over IPC and sends their source to the renderer, which loads it through a Blob URL and `import()`.
Users can disable them in settings, and remove them by deleting their folder.

## Manifest

Each extension has a `cushion-extension.json`:

```jsonc
{
  "id": "cushion.csv-viewer",
  "name": "CSV Viewer",
  "version": "0.1.0",
  "description": "View and edit CSV files as a table",
  "fileTypes": [
    { "extensions": ["csv"] }
  ],
  "main": "main.js"
}
```

| Field | Required | Description |
|---|---|---|
| `id` | yes | Unique identifier (e.g. `cushion.csv-viewer`) |
| `name` | yes | Human-readable name |
| `version` | yes | Semver version |
| `fileTypes` | yes | Array of file type bindings |
| `fileTypes[].extensions` | yes | File extensions without the dot (e.g. `["csv"]`) |
| `fileTypes[].binary` | no | If `true`, the file is read as base64. Default `false` |
| `main` | external only | Pre-bundled ESM `.js` file at the top of the extension directory. Bundled extensions omit it |
| `description` | no | Short description |
| `author` | no | Author name |
| `icon` | no | SVG file name at the top of the extension directory |
| `cushionVersion` | no | Semver range for compatibility |

## Extension API

An extension receives one `ExtensionContext` object (`packages/extension-api/src/context.ts`), its only interface to Cushion.

```typescript
interface ExtensionContext {
  filePath: string;
  file: ExtensionFileAPI;
  theme: 'light' | 'dark';
  isActive: boolean;
}

interface ExtensionFileAPI {
  read(): Promise<string>;
  readChunk(offset: number, length: number): Promise<ChunkResult>;
  update(content: string): void;
  updateBase64(base64: string): void;
  flush(): Promise<void>;
  onExternalChange(callback: (content: string | null) => void): () => void;
}
```

Extensions get no access to other files, CodeMirror, Zustand stores, or internal React context.
They cannot add sidebar panels, commands, or keybindings.

External extensions default-export a React component:

```tsx
import type { ExtensionContext } from '@cushion/extension-api';

export default function MyViewer({ ctx }: { ctx: ExtensionContext }) {
  return <div>...</div>;
}
```

## Save pipeline

Cushion owns the save pipeline, and extensions never write to disk.

1. The extension calls `ctx.file.update(content)` or `ctx.file.updateBase64(base64)`.
2. `ExtensionHost` debounces for 1s, writes to disk, and remembers the last written content.
3. On a file watcher event, `ExtensionHost` rereads the file and calls the `onExternalChange` callbacks, unless the content matches its own last write.
   Binary files get `null`.
4. Ctrl+S or Cmd+S inside the extension pane flushes the pending write.
   `flush()` does the same from code, and PDF annotations use it.
5. Unmounting the host flushes any pending write.

Read-only extensions never call `update()`.

## Per-workspace config

Each workspace can disable extensions in `.cushion/extensions.json`:

```jsonc
{
  "disabled": ["cushion.csv-viewer"]
}
```

## Loading

Bundled extensions register in `registerBuiltinViews()` (`register-builtin-views.ts`).
It imports the manifests through the `@extensions` Vite alias, imports the components directly, and skips IDs in the `disabled` set.
Calling it again with a new set syncs the registry without a page reload.

External extensions load from `Home.tsx` on workspace change:

1. `loadGlobalExtensions()` reads `.cushion/extensions.json` when a workspace is open.
2. The coordinator scans `~/.cushion/extensions/` and validates each manifest with `parseManifest()`.
3. The loader filters out disabled extensions and skips any without `main`.
4. Each extension registers through `registerExternalExtensionView()` with a `React.lazy()` component.
5. On first open, the loader fetches the source over IPC, strips React imports, prepends a shim that reads `window.__CushionReact`, and imports it from a Blob URL.

External extensions must ship as pre-bundled ESM `.js` with React marked external.

## View registry

`view-registry.ts` holds a discriminated union with two sources:

| Source | Registration | Cleanup |
|---|---|---|
| `'bundled'` | `registerBundledView()` | `unregisterView()` |
| `'extension'` | `registerExternalExtensionView()` | `unregisterExternalViews()` |

Both render `{ ctx: ExtensionContext }` through `ExtensionHost`.
`unregisterView(id)` removes one view, and `unregisterExternalViews()` removes every external one.

`isBinaryFile()` checks a set built from the registered manifests.
The set starts with the known binary types (png, jpg, pdf and so on), and unregistering a view never removes those, so disabling a viewer keeps binary detection.

`EditorPane` picks a view in this order: registered view, then markdown (`CodeEditor`), then a "No viewer available" message for binary files, then `MonacoEditor` for everything else.

## ExtensionHost

`ExtensionHost.tsx` wraps every extension view with:

- `ExtensionErrorBoundary`, so a broken extension can't crash the app
- `Suspense` with a loading message for lazy components
- the save pipeline above

## Settings UI

`ExtensionsSettings.tsx` is the Settings → Extensions tab.
It lists bundled and external extensions with search, and shows name, version, description, file types and a Bundled/External badge.
Toggling an extension writes `.cushion/extensions.json`, calls `unloadExternalExtensions()` and `loadGlobalExtensions()`, and passes the new set to `registerBuiltinViews()`.
The toggle needs an open workspace, because the disabled list is per workspace.

## Coordinator handlers

`handlers/extensions.ts` serves three IPC methods:

```
extensions/discover-global    scan ~/.cushion/extensions/
extensions/read-source        return the extension's JS (5MB cap, needs manifest.main, path-sanitized)
extensions/read-icon          return the SVG icon (path-sanitized)
```

## File structure

```
packages/extension-api/
  src/
    context.ts, manifest.ts, validate-manifest.ts, index.ts
  manifests/
    cushion-image/cushion-extension.json
    cushion-pdf/cushion-extension.json
    cushion-excalidraw/cushion-extension.json

apps/frontend/
  lib/
    view-registry.ts
    extension-loader.ts
    register-builtin-views.ts
  components/editor/
    ExtensionHost.tsx
    ExtensionErrorBoundary.tsx
    ImageExtension.tsx
    PdfExtension.tsx
    ExcalidrawExtension.tsx
  components/settings/
    ExtensionsSettings.tsx

apps/electron/src/main/coordinator/handlers/extensions.ts

~/.cushion/extensions/        global user extensions
.cushion/extensions.json      per-workspace disabled list
.cushion/attachments/         pasted image attachments
```

## Decisions

### Built-in features are extensions

Image, PDF and Excalidraw use the same API and manifests as external extensions, like VS Code's built-in extensions.
Obsidian and Logseq do the opposite: their core features are compiled in and use internal APIs that plugins can't reach.
VS Code goes further and gives built-ins about 40 "proposed APIs" through a `product.json` allowlist.
Cushion has no privilege tiers, so bundled and external extensions have the same capabilities.

### Two loading paths

Loading everything over IPC would cost too much on first open.
CSV is 4.4 KB, pdfjs is 0.72 MB, and Excalidraw is 7.87 MB.
Vite imports give bundled extensions code splitting and tree shaking with no IPC round trip.

### Same renderer, no isolation

Extensions run in the renderer, in a div, without a separate process or iframe.

- VS Code runs extensions in a separate Node.js process with no DOM, and serializes every API call over JSON-RPC.
  A bad extension can't take down the UI.
- Logseq uses iframes for community plugins, with a postMessage handshake (up to 5 retries at 500ms, 8s timeout) that adds 50 to 100ms per mount.
- Obsidian runs plugins in the renderer through `eval()` with full DOM and filesystem access.

Cushion picks latency over isolation.
An extension that hangs blocks the main thread, which is acceptable while there are few extensions.

### Loading method doesn't change security

`eval()`, Blob URL `import()` and direct imports are equivalent in a renderer with `contextIsolation: true` and `nodeIntegration: false`.
The security boundary is what `contextBridge` exposes over IPC.
Extensions can touch the DOM and prototypes whatever the loading method, and real isolation would need iframes on separate origins.
Cushion uses Blob URL `import()` because the extension loads as a real ES module.
`resolveExtensionDir()` runs `path.basename()` on the directory name to block path traversal over IPC.

### `main` is optional in the schema

Bundled extensions have no entry point to fetch, so they omit `main`.
`extension-loader.ts` enforces it for external extensions; the schema validator doesn't.

### `@extensions` Vite alias

It points at `packages/extension-api/manifests/` and is set in both `apps/frontend/vite.config.ts` and `apps/electron/electron.vite.config.ts`, so manifest imports work in dev and in packaged builds.

### File types live in the manifest

Obsidian plugins register file types in `onload()`, so you have to run a plugin to learn what it handles.
Cushion reads `fileTypes` from the manifest without running extension code.

### Extensions are global

Extension code lives only in `~/.cushion/extensions/`, never in the vault.
Workspaces only choose what to disable, through `.cushion/extensions.json`, much like disabling a VS Code extension in a workspace's `settings.json`.

### Attachments go to `.cushion/attachments/`

Pasted images go to `.cushion/attachments/` at the vault root, not to per-note `.attachments/` folders.
Old `![[path/.attachments/...]]` embeds still resolve, so there's no migration.
`listAllFilePaths()` sees the folder (it isn't in `IGNORED_PATTERNS`), and the file browser hides it because it starts with a dot.

## Open questions

- Hot reload for extensions during development.
- Extension-declared settings.
- API versioning finer than the `cushionVersion` semver range.
- Extension icons in the tab bar (`PaneTabBar.tsx`).
