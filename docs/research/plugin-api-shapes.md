# How Obsidian and VS Code shape their plugin APIs

Research for [#52](https://github.com/Aleexc12/cushion/issues/52), part of the plugin system map [#48](https://github.com/Aleexc12/cushion/issues/48).
Researched on 2026-10-05 against Obsidian 1.14 (npm `obsidian` 1.14.4) and the current VS Code `vscode.d.ts`.
VS Code calls its plugins "extensions", and this file keeps that word only when it quotes VS Code.
Cushion's own term is "plugin".

## Short answer

The two APIs sit at opposite ends.
Obsidian is all runtime: the manifest only identifies the plugin, and every command, view, setting tab and editor hook gets registered by code in `onload()`.
VS Code is manifest first: `package.json` declares commands, menus, keybindings, settings, views and editors statically, and code only supplies the implementations once an activation event fires.

Both have arrived at the same lifecycle.
Every registration returns something disposable, the host tracks it per plugin, and unloading the plugin cleans everything up.

Neither has a permission system.
Both rely on distribution controls, meaning review, scanning and signing, plus a global "run no plugins" mode.
Obsidian announced in May 2026 that plugins will declare the capabilities they use, but only as disclosures and not enforced.

For Cushion's API v1 the best thing to copy is VS Code's split between static declarations and runtime implementations, kept to a handful of contribution points.
From Obsidian, copy the small `Plugin` base class with auto-cleaned `register*` helpers, the raw CodeMirror 6 hook, and its late, painful lessons: deferred views, declarative settings, and never handing out view references or core DOM.

## Contribution points compared

| Contribution | Obsidian (runtime, in `onload`) | VS Code (manifest + runtime) |
|---|---|---|
| Commands | `addCommand({id, name, callback / checkCallback / editorCallback / editorCheckCallback, hotkeys?})` | `contributes.commands` (`command`, `title`, `category`, `icon`, `enablement`) + `commands.registerCommand(id, fn): Disposable` |
| Keybindings | `Command.hotkeys` defaults (discouraged) or `Scope.register(mods, key, fn)` on a view's `scope` | `contributes.keybindings` (`command`, `key`, `mac`, `when`) |
| Context menus | Workspace events `file-menu`, `files-menu`, `editor-menu`, `url-menu`; plugin adds items to a `Menu` | `contributes.menus` keyed by menu ID (`editor/context`, `explorer/context`, `view/title`, ...) with `when` and `group` |
| Sidebar panels and views | `registerView(type, leaf => new MyView(leaf))`; `ItemView` gets a raw `containerEl`; opened via `workspace.getRightLeaf()` / `ensureSideLeaf()` | `contributes.viewsContainers` + `contributes.views` + `registerTreeDataProvider` or `registerWebviewViewProvider` |
| File-type viewers and editors | `registerExtensions(['csv'], viewType)` + `FileView` / `TextFileView` (`getViewData`, `setViewData`, debounced `requestSave`) | `contributes.customEditors` (`viewType`, `selector` globs, `priority`) + `registerCustomEditorProvider`; text variant reuses the host document, save and undo |
| Settings UI | `addSettingTab(PluginSettingTab)`; imperative `new Setting(el)` builder; declarative `getSettingDefinitions()` since 1.13.0 | `contributes.configuration` as JSON Schema; host renders it in the Settings editor |
| Status bar | `addStatusBarItem(): HTMLElement` (desktop only) | Runtime only: `window.createStatusBarItem(id, alignment, priority)` returns a `StatusBarItem` object |
| Ribbon / activity bar | `addRibbonIcon(icon, title, cb): HTMLElement` | `viewsContainers.activitybar` entry |
| Editor extensions | `registerEditorExtension(cm6Extension)`, host shares one CM6 instance; `registerEditorSuggest` | Language features through provider registries (`languages.register*Provider`); no raw editor access |
| Rendered markdown | `registerMarkdownPostProcessor`, `registerMarkdownCodeBlockProcessor` (Reading view only) | Built-in preview only: `markdown.markdownItPlugins`, `markdown.previewStyles`, `markdown.previewScripts` |
| Modals and notices | `Modal`, `SuggestModal`, `FuzzySuggestModal`, `ConfirmationModal`, `new Notice(msg)` | `window.showQuickPick`, `showInputBox`, `showInformationMessage` (host-rendered, no custom DOM) |
| Deep links | `registerObsidianProtocolHandler(action, fn)` | `window.registerUriHandler` + `onUri` activation |
| Newer points | `registerBasesView` (1.10), `registerCliHandler` (1.12.2), `registerHoverLinkSource` | 38 manifest points in total, including `walkthroughs`, `viewsWelcome`, chat and language-model tools |

The pattern that matters for Cushion is the return type.
Obsidian hands back raw DOM for its UI points (`HTMLElement` for the status bar and ribbon, `containerEl` for views).
VS Code hands back objects (`StatusBarItem`, `TreeView`) or renders host UI from data (quick picks, tree items, settings).
Data-driven points let the host restyle, move or virtualize UI without breaking plugins.
Raw-DOM points freeze the host's markup into the API.

## Lifecycle

### Obsidian

`Plugin extends Component`.
`Component` provides `onload`, `onunload`, `addChild`, `register(cb)`, `registerEvent(ref)`, `registerDomEvent(el, type, cb)` and `registerInterval(id)`.
Everything registered through these, and everything added with `Plugin.add*` / `register*`, is removed automatically when the plugin unloads.
Children load after their parent and unload before it.
A common bug is creating an orphan `new Component()` to pass to `MarkdownRenderer.render`, which then never unloads; the official lint rule `no-plugin-as-component` exists for it.

Extra hooks have accumulated:
- `workspace.onLayoutReady(cb)` for work that should wait until startup is done.
- `onExternalSettingsChange()` (1.5.7) when `data.json` changes on disk, for example through Sync.
- `onUserEnable()` (1.7.2), run once when the user turns the plugin on, so first-run UI like opening a view does not happen on every launch.

There is no lazy activation.
"Obsidian loads all plugins before the user can interact with the app", so the load-time guide asks plugins to do only registrations in `onload` and to defer everything else.

### VS Code

The entry module exports `activate(context)` and optionally `deactivate()`.
Every registration returns a `Disposable`, and the plugin pushes it onto `context.subscriptions`; the host disposes the lot on deactivation.
`deactivate` must return a promise if cleanup is async.

Activation is lazy and event-driven: `onCommand`, `onView`, `onLanguage`, `onCustomEditor`, `workspaceContains`, `onUri`, `onStartupFinished` and `*`, among others.
Since 1.74 the host derives `onCommand`, `onView`, `onLanguage`, `onCustomEditor` and `onAuthenticationRequest` from the contributions, so most plugins no longer list activation events at all.

Plugin code runs in a separate extension host process, an Electron utility process since 1.75.
The host can crash and restart without taking down the window; since 1.16 the workbench removes plugin-driven UI on a crash and restores it on restart.
Since 1.88 updating a plugin restarts plugins without reloading the window.

## Settings and data storage

| Need | Obsidian | VS Code |
|---|---|---|
| Settings schema | Plugin-defined object; since 1.13.0 `getSettingDefinitions()` returns `{name, desc, control: {type, key, defaultValue, validate}}` items, types include toggle, dropdown, text, number, file, folder, slider, color, secret | `contributes.configuration` JSON Schema with `default`, `enum`, `markdownDescription`, `scope` (`application`, `machine`, `window`, `resource`, `language-overridable`) |
| Read and write settings | `this.settings`; host calls `saveData()` for declarative settings | `workspace.getConfiguration(section).get / update`, `onDidChangeConfiguration` with `affectsConfiguration` |
| Plugin data | `loadData()` / `saveData()` on one `data.json` in `<vault>/.obsidian/plugins/<id>/`, per vault | `globalState` and `workspaceState` key-value Mementos; `globalStorageUri` / `storageUri` folders for large files |
| Sync | Whole `data.json` syncs with the vault | `globalState.setKeysForSync(keys)` (1.51); settings can opt out with `ignoreSync` (1.75) |
| Secrets | `app.secretStorage` (1.11.4), vault-scoped local storage, shared across plugins: settings store a secret's name, any plugin can `listSecrets()` | `context.secrets` (1.53), per plugin, encrypted, never synced |
| Small UI state | `app.loadLocalStorage` / `saveLocalStorage` (1.8.7), vault-scoped | `workspaceState` |

Obsidian's move to declarative settings in 1.13.0 is the most relevant change here.
The stated reason was global settings search: the host can only index settings it can read without running the plugin's `display()` code.
`SettingTab.display()` is now deprecated.
VS Code had this from the start because settings were always JSON Schema.

## Manifest versus runtime registration

### Obsidian's manifest identifies, nothing more

`manifest.json` requires `id`, `name`, `version`, `minAppVersion`, `description`, `author` and `isDesktopOnly`, plus optional `authorUrl` and `fundingUrl`.
`versions.json` maps plugin versions to minimum app versions so an old app can install the newest compatible release.
At runtime, `requireApiVersion(v)` gates newer API calls, and the lint rule `no-unsupported-api` checks calls against `minAppVersion`.

Nothing in the manifest declares commands, views, settings or file types.
To know what a plugin contributes, the host has to run it.
Cushion's current file viewer system already chose the other way and declares `fileTypes` in its manifest for this reason.

### VS Code's manifest declares the UI

`package.json` requires `name`, `version`, `publisher` and `engines.vscode` (a semver range, never `*`).
`contributes` holds the static UI, `activationEvents` the remaining triggers, and `capabilities` the trust support (`untrustedWorkspaces`, `virtualWorkspaces`).
`extensionDependencies` and `extensionPack` express plugin-to-plugin dependencies and groups.

The extension host docs give the reason for the split: avoid hurting startup and UI performance, so the Markdown plugin only loads when a Markdown file opens.
Because declarations are data, the command palette, menus, keybinding editor, settings editor and welcome views can show plugin entries before the plugin runs, and invoking one activates it.

### API staging

VS Code ships new API as "proposed" first: `vscode.proposed.<name>.d.ts` files, opted into with `enabledApiProposals`, usable only in Insiders or with a flag, and not publishable to the Marketplace.
Finalization asks for a non-trivial sample and multiple distinct use cases.
Obsidian has no staging tier; new API lands in the typings with an `@since` tag.

## Regrets and breaking changes

### Obsidian

- **Deferred views (1.7.2).**
  Restored views no longer load until their tab becomes visible.
  `leaf.view` may now be a `DeferredView`, so plugins that cast after `getViewType()` or kept references to views broke.
  The guidance now is: never keep view references, check with `instanceof`, and `await workspace.revealLeaf(leaf)` before talking to a view.
- **Declarative settings (1.13.0).**
  The imperative `display()` builder could not be searched, so the API grew a second, declarative way and deprecated the first.
- **CodeMirror 5 to 6.**
  CM6 became the default in 0.13, but the legacy editor stayed until 1.5 because "many of Obsidian's first community plugins relied on the legacy editor".
  The `Editor` abstraction still carries CM5-era methods.
  Exposing a raw CM6 `Extension` couples plugins to the host's CodeMirror version; the 1.13.0 release notes list a CodeMirror upgrade among the breaking changes.
- **Global state.**
  The guidelines tell plugins to use `this.app` instead of the global `app`, which "might be removed".
- **Popout windows (0.15).**
  Plugins assumed one global `document`, so the API grew `activeDocument`, `el.doc` and `el.win`, `instanceOf` helpers and `onWindowMigrated`.
- **Startup cost.**
  Since every plugin loads before the user can interact, a whole guide asks authors to keep `onload` to registrations and move work into `onLayoutReady`.
- **Default hotkeys.**
  Discouraged in the typings, the guidelines and the lint rules, because plugin defaults collide with each other and with user bindings.
- **Mobile.**
  Node and Electron APIs crash on mobile, hence `isDesktopOnly` and `Platform` checks.
  This one does not apply to Cushion.
- **Security.**
  "Obsidian cannot reliably restrict plugins to specific permissions or access levels."
  The mitigations are Restricted mode by default, no auto-updates, an automated scanner on every version since 2026, manual review for popular or flagged plugins, and planned capability disclosures.

### VS Code

- **Activation with `*`.**
  Plugins activating at startup slowed everyone down.
  1.46 added `onStartupFinished`, and the docs ask authors to use `*` only when nothing else works.
- **Listing activation events by hand.**
  1.74 made the host derive them from contributions, so the manifest stopped repeating itself.
- **`workspace.rootPath`.**
  A single-folder assumption, replaced by `workspaceFolders` for multi-root workspaces.
- **`vscode.previewHtml`.**
  Removed in 1.33 for "security and compatibility issues that we determined could not be fixed without breaking existing users"; webviews replaced it.
- **The `vscode` npm module.**
  Split in 1.36 into `@types/vscode` and a test package, after the event-stream incident showed its 223 transitive dependencies were a supply-chain risk.
- **Webviews.**
  The docs say to use them "sparingly and only when VS Code's native API is inadequate", and the UX guidelines forbid promotional panels and views that open on every launch.
- **No DOM, by design.**
  "Extensions have no access to the DOM of VS Code UI."
  This is the decision that let VS Code move the extension host out of the renderer in 2022 without breaking plugins.
- **API stability.**
  The API guidelines open with "We DO NOT want to break API."
  They prescribe `on[Did|Will]VerbSubject` events on namespaces, synchronous `createXYZ` factories, sync reads and async writes, a `CancellationToken` on every provider call, and "shy" objects that expose only what the API defines.
- **Security.**
  "The extension host has the same permissions as VS Code itself."
  Controls live at the distribution layer: signed uploads verified on install since 1.75, malware scanning, verified publishers, a block list that uninstalls flagged plugins, a publisher-trust prompt since 1.97, and Workspace Trust (1.57) for untrusted folders.

## What this suggests for Cushion's API v1

These are recommendations for the grilling tickets under #48, not decisions.

1. **Declare statically, implement at runtime.**
   The manifest lists commands (id, title, icon, default keybinding), file viewers (file extensions, binary flag), panels (id, title, icon, side), settings (schema) and menu entries.
   Code registers the matching implementations through `activate(ctx)`.
   This keeps the existing `fileTypes` decision, makes the command palette, shortcuts settings and global settings search work without running plugin code, and makes lazy activation possible.
2. **Derive activation from contributions.**
   Do not ask authors for activation events.
   Activate a plugin the first time one of its declared commands runs, one of its file types opens, or one of its panels is shown, with an explicit "on startup" opt-in for the rest.
   This is VS Code after 1.74, and it avoids Obsidian's "every plugin loads before the user can type" cost.
3. **One disposable lifecycle.**
   `activate(ctx)` / `deactivate()`, every `ctx.*.register*` returns a disposable, and the host tracks and disposes all of them per plugin.
   No `Component` tree for plugin authors to get wrong.
4. **Data-driven UI points, React only inside owned containers.**
   Status bar items, menu items, notices and pickers should be objects or data the core renders, not DOM handed to the plugin.
   Panels and file viewers mount a plugin's React component into a container the core owns, which is what the current file viewer host already does.
   Never expose core DOM or Zustand stores.
5. **Declarative settings rendered by the core.**
   A small JSON Schema subset in the manifest, read and written through `ctx.settings`, with change events.
   Separate `ctx.storage` (global and per-workspace key-value) and `ctx.secrets` scoped per plugin and backed by Electron `safeStorage`, so one plugin cannot list another's secrets as it can in Obsidian.
6. **Keep the API serializable where it can be.**
   Promise-returning calls with plain data arguments keep the door open to moving plugin code out of the renderer later, as VS Code did in 2022.
   The React rendering of panels and viewers is the one part that cannot be serialized.
7. **CodeMirror access as a deliberate, versioned exception.**
   WYSIWYG plugins need raw CM6 extensions, and Obsidian shows that works when the host shares a single CM6 instance as a build external.
   It also shows the cost: every CodeMirror upgrade is a potential break, so mark this part of the API as tied to the CodeMirror version.
8. **Version and stage the API from day one.**
   An `engines.cushion` semver range in the manifest, an exported API version for runtime checks, and a clearly separated unstable namespace for API that is still settling.
   Whether built-in plugins may use unstable API conflicts with the existing "no privilege tiers" decision and needs a call.
9. **Default keybindings: allow, but the user wins.**
   Obsidian discourages them; VS Code allows them with `when` clauses.
   With a curated registry, declared defaults plus conflict display in the shortcuts settings seems the better trade.
10. **Leave room for capability declarations.**
    Neither app enforces permissions, and Obsidian is now adding declared capabilities after the fact.
    If every core call goes through `ctx` rather than `window.electronAPI`, a manifest `capabilities` field can later be shown at install time and even enforced.
    This belongs to the trust model ticket.

## Things not verified

- The exact Obsidian version that removed the CM5 hook `registerCodeMirror`.
- The 1.12 Obsidian CLI release wording, seen only in a search snippet.
- The 2025 Visual Studio Marketplace removal figures, because the devblogs post redirect-loops.
- A VS Code team post explaining the no-DOM decision beyond the capabilities overview; `/blogs/2019/10/03/extensions-and-the-ui` returns 404.
- The VS Code version that deprecated `workspace.rootPath` (likely 1.18).
- Obsidian's typings mark `isDesktopOnly` optional and omit `fundingUrl`, while the manifest docs list `isDesktopOnly` as required.

## Sources

### Obsidian

- Typings, `obsidian.d.ts` (npm `obsidian` 1.14.4): https://github.com/obsidianmd/obsidian-api/blob/master/obsidian.d.ts
- API changelog (ends at 1.7.2): https://github.com/obsidianmd/obsidian-api/blob/master/CHANGELOG.md
- Developer docs source: https://github.com/obsidianmd/obsidian-developer-docs
- Views: https://docs.obsidian.md/Plugins/User+interface/Views
- Commands: https://docs.obsidian.md/Plugins/User+interface/Commands
- Context menus: https://docs.obsidian.md/Plugins/User+interface/Context+menus
- Settings: https://docs.obsidian.md/Plugins/User+interface/Settings
- Migrate to declarative settings: https://docs.obsidian.md/Plugins/Guides/Migrate+to+declarative+settings
- Editor extensions: https://docs.obsidian.md/Plugins/Editor/Editor+extensions
- Manage plugin lifecycle: https://docs.obsidian.md/Plugins/Guides/Manage+plugin+lifecycle
- Optimize plugin load time: https://docs.obsidian.md/Plugins/Guides/Optimize+plugin+load+time
- Defer views: https://docs.obsidian.md/Plugins/Guides/Defer+views
- Support pop-out windows: https://docs.obsidian.md/Plugins/Guides/Support+pop-out+windows
- Store secrets: https://docs.obsidian.md/Plugins/Guides/Store+secrets
- `onUserEnable`: https://docs.obsidian.md/Reference/TypeScript+API/Plugin/onUserEnable
- `registerCliHandler`: https://docs.obsidian.md/Reference/TypeScript+API/Plugin/registerCliHandler
- Manifest: https://docs.obsidian.md/Reference/Manifest
- Plugin guidelines: https://docs.obsidian.md/Plugins/Releasing/Plugin+guidelines
- Submission requirements: https://docs.obsidian.md/Community+directory/Submission+requirements+for+plugins
- Developer policies: https://docs.obsidian.md/Developer+policies
- Mobile development: https://docs.obsidian.md/Plugins/Getting+started/Mobile+development
- Official ESLint plugin: https://github.com/obsidianmd/eslint-plugin
- Sample plugin: https://github.com/obsidianmd/obsidian-sample-plugin
- Releases repo and `community-plugins.json`: https://github.com/obsidianmd/obsidian-releases
- Plugin security help page: https://obsidian.md/help/plugin-security
- Community plugins help page: https://obsidian.md/help/community-plugins
- 1.7.2 changelog: https://obsidian.md/changelog/2024-09-19-desktop-v1.7.2/
- 1.13.0 changelog: https://obsidian.md/changelog/2026-05-28-desktop-v1.13.0/
- Goodbye legacy editor (2023-11-02): https://obsidian.md/blog/goodbye-legacy-editor/
- The future of plugins (kepano, 2026-05-12): https://obsidian.md/blog/future-of-plugins/

### VS Code

- Typings, `vscode.d.ts`: https://github.com/microsoft/vscode/blob/main/src/vscode-dts/vscode.d.ts
- Extension anatomy: https://code.visualstudio.com/api/get-started/extension-anatomy
- Contribution points: https://code.visualstudio.com/api/references/contribution-points
- Activation events: https://code.visualstudio.com/api/references/activation-events
- Extension manifest: https://code.visualstudio.com/api/references/extension-manifest
- When clause contexts: https://code.visualstudio.com/api/references/when-clause-contexts
- API reference and patterns: https://code.visualstudio.com/api/references/vscode-api
- Extension host: https://code.visualstudio.com/api/advanced-topics/extension-host
- Capabilities overview and restrictions: https://code.visualstudio.com/api/extension-capabilities/overview
- Common capabilities: https://code.visualstudio.com/api/extension-capabilities/common-capabilities
- Commands guide: https://code.visualstudio.com/api/extension-guides/command
- Tree views: https://code.visualstudio.com/api/extension-guides/tree-view
- Webviews: https://code.visualstudio.com/api/extension-guides/webview
- Custom editors: https://code.visualstudio.com/api/extension-guides/custom-editors
- Markdown preview extension guide: https://code.visualstudio.com/api/extension-guides/markdown-extension
- Workspace Trust guide: https://code.visualstudio.com/api/extension-guides/workspace-trust
- Remote extensions: https://code.visualstudio.com/api/advanced-topics/remote-extensions
- Using proposed API: https://code.visualstudio.com/api/advanced-topics/using-proposed-api
- Extension API process (wiki): https://github.com/microsoft/vscode/wiki/Extension-API-process
- Extension API guidelines (wiki): https://github.com/microsoft/vscode/wiki/Extension-API-guidelines
- Multi-root workspace APIs (wiki): https://github.com/microsoft/vscode/wiki/Adopting-Multi-Root-Workspace-APIs
- UX guidelines, status bar: https://code.visualstudio.com/api/ux-guidelines/status-bar
- UX guidelines, webviews: https://code.visualstudio.com/api/ux-guidelines/webviews
- UX guidelines, views: https://code.visualstudio.com/api/ux-guidelines/views
- Bundling extensions: https://code.visualstudio.com/api/working-with-extensions/bundling-extension
- Publishing extensions: https://code.visualstudio.com/api/working-with-extensions/publishing-extension
- Extension runtime security: https://code.visualstudio.com/docs/configure/extensions/extension-runtime-security
- Extension Marketplace: https://code.visualstudio.com/docs/configure/extensions/extension-marketplace
- Release notes 1.16, 1.33, 1.36, 1.46, 1.51, 1.53, 1.57, 1.74, 1.75, 1.88, 1.97: https://code.visualstudio.com/updates/v1_16 (replace the version in the URL)
- Sandboxing VS Code (2022-11-28): https://code.visualstudio.com/blogs/2022/11/28/vscode-sandbox
- Workspace Trust blog (2021-07-06): https://code.visualstudio.com/blogs/2021/07/06/workspace-trust
- event-stream incident (2018-11-26): https://code.visualstudio.com/blogs/2018/11/26/event-stream
- Extension bisect (2021-02-16): https://code.visualstudio.com/blogs/2021/02/16/extension-bisect
