# How can a plugin run code in the Electron main process?

Research for [#51](https://github.com/Aleexc12/cushion/issues/51), part of map [#48](https://github.com/Aleexc12/cushion/issues/48).
Target is Windows x64 on Electron 35 (Node 22.14, Chromium 134) ([release post](https://www.electronjs.org/blog/electron-35-0)).

## Short answer

Give each plugin that needs Node an optional Node entry point, and run it in its own Electron `utilityProcess`, started on demand by the coordinator.
The plugin's renderer half and Node half talk over a `MessagePort` that the main process brokers once and then stays out of.
The Node half has full Node, so it can spawn its own binaries with `child_process`, but those binaries should be spawned through a core API so the coordinator owns their lifetime.
Do not load plugin code into the main process.

A process boundary buys crash isolation and a responsive main process.
It buys no security, because Electron's utility process is not sandboxed.
Security has to come from the trust model (#50) and registry review.

## Where Cushion is today

- The renderer runs with `contextIsolation: true`, `nodeIntegration: false` and the default sandbox (`apps/electron/src/main/index.ts`).
- The preload exposes one generic `invoke(method, params)` that maps to `coordinator:${method}`, so any renderer code, including external plugin code, can call any coordinator method (`apps/electron/src/preload/index.ts`).
- The coordinator already spawns child processes from main with `child_process`: the Sherpa WebSocket server (`sherpa-manager.ts`), Pandoc (`handlers/pandoc.ts`), opencode (`opencode-server.ts`) and an allowlisted `shell/exec` for NotebookLM (`handlers/shell.ts`).
- Downloaded binaries already live outside the asar in `%APPDATA%/Cushion/bin/<name>` (`app.getPath('userData')`), and opencode is listed under `asarUnpack` in `electron-builder.yml`.
- Sherpa shutdown already uses `taskkill /pid <pid> /f /t` to kill the whole process tree on Windows.

## Options in Electron 35

### 1. Load plugin modules into the main process

The coordinator would `require()` the plugin's Node entry and let it register IPC handlers.

- Isolation: none. The plugin shares the main process's memory, globals and event loop.
  Electron's security checklist warns against processing untrusted content in any unsandboxed process, "including the main process" ([security.md #4](https://github.com/electron/electron/blob/35-x-y/docs/tutorial/security.md)).
- Responsiveness: a plugin that blocks the event loop stalls every `ipcMain` handler, window management and the file watcher, so the whole app freezes.
- Crash behaviour: an uncaught JS exception shows Electron's "A JavaScript error occurred in the main process" dialog and does not quit ([lib/browser/init.ts](https://github.com/electron/electron/blob/35-x-y/lib/browser/init.ts)).
  A native crash, an out-of-memory error or `process.exit()` in plugin code ends the whole app, since main owns the application lifecycle ([process model](https://github.com/electron/electron/blob/35-x-y/docs/tutorial/process-model.md)).
  That last point is inferred; I found no doc sentence that says it outright.
- Disable and uninstall: Node cannot unload a module, so disabling a plugin cleanly means restarting the app.
- IPC wiring: the cheapest of all options, since plugin code calls core functions directly.
- Packaging: plugin JS can load from any absolute path under `userData`; native modules need the Electron ABI (see below).
- `node:vm` does not fix the isolation problem: "The `node:vm` module is not a security mechanism. Do not use it to run untrusted code." ([Node docs](https://nodejs.org/docs/latest-v22.x/api/vm.html)).

### 2. `utilityProcess` per plugin, or one shared plugin host

`utilityProcess.fork(modulePath, args, options)` starts a Chromium child process running Node with Electron's Node build ([utility-process.md](https://github.com/electron/electron/blob/35-x-y/docs/api/utility-process.md)).

- API: can only be called after `app` `ready`.
  Options include `env`, `execArgv`, `cwd`, `serviceName` and `stdio` (stdout and stderr only, stdin is always ignored).
  The process object has `postMessage(message, [ports])`, `kill()`, `pid`, `stdout`, `stderr` and the events `spawn`, `message`, `exit` (with code) and an experimental `error`.
  `allowLoadingUnsignedLibraries` is macOS only and irrelevant here.
- Intended use: Electron's process model doc recommends it for "untrusted services, CPU intensive tasks or crash prone components" and says apps should prefer it over `child_process.fork` because it can open a `MessagePort` channel to a renderer ([process model](https://github.com/electron/electron/blob/35-x-y/docs/tutorial/process-model.md)).
- Isolation: separate process, separate memory, separate event loop.
  It is not sandboxed: the Electron source says "Currently utility process is not sandboxed" ([electron_api_utility_process.cc](https://github.com/electron/electron/blob/35-x-y/shell/browser/api/electron_api_utility_process.cc)).
  The plugin can read any file, open sockets and spawn processes, exactly like main.
- Crash behaviour: main survives.
  The process fires `exit` with its code, and `app` fires `child-process-gone` with `type: 'Utility'`, a `reason` (`crashed`, `oom`, `abnormal-exit`, `killed`, ...), `exitCode` and `serviceName` ([app.md](https://github.com/electron/electron/blob/35-x-y/docs/api/app.md)).
  `serviceName` shows up in `app.getAppMetrics()`, so per-plugin CPU and memory are visible for free.
- Disable and uninstall: `kill()` the process. No app restart needed.
- IPC wiring: two channels.
  The plugin talks to core over `process.parentPort` (main side: `child.postMessage` / `child.on('message')`).
  For renderer traffic, main creates a `MessageChannelMain`, sends one port to the utility process and one to the window, and is then out of the loop ([message-ports.md](https://github.com/electron/electron/blob/35-x-y/docs/tutorial/message-ports.md)).
  Ports only travel through `postMessage` variants, never `ipcRenderer.invoke`, so the preload needs a small addition that receives the port and forwards it into the context-isolated main world with `window.postMessage(..., [port])`, as the message-ports tutorial shows.
  Cushion would need an RPC layer on top (request ids, replies, errors, cancellation); VS Code's `RPCProtocol` is the reference design.
- Packaging: the plugin's entry file lives on disk under `userData`, outside the asar, so asar limits do not apply to community plugins.
  For built-in plugins whose entry sits inside `app.asar`, check that `utilityProcess.fork` accepts an asar path before relying on it.
- Cost: one extra Chromium process per running plugin.
  I did not find an official memory figure; measure it on Windows before committing to one process per plugin.

### 3. Plain `child_process`

`spawn` / `execFile` / `fork` from main, as the coordinator does today for Sherpa, Pandoc and opencode.

- Right tool for a plugin's external binaries (Sherpa, Pandoc, Python for NotebookLM).
  It is the wrong tool for plugin JS: `child_process.fork` of plugin JS would need `ELECTRON_RUN_AS_NODE` and gives no `MessagePort` to the renderer, which is why Electron points to `utilityProcess` instead ([process model](https://github.com/electron/electron/blob/35-x-y/docs/tutorial/process-model.md)).
- Isolation and crash behaviour: separate OS process, main survives a crash, exit code via `close` / `exit`.
- IPC wiring: stdio pipes, a local socket or a local WebSocket (what Sherpa does today).
- Packaging: binaries cannot run from inside an asar with `spawn` or `exec`; only `execFile` works there, by extracting to a temp file, and `cwd` cannot point inside an asar ([asar-archives.md](https://github.com/electron/electron/blob/35-x-y/docs/tutorial/asar-archives.md)).
  Downloading binaries to `userData` at install time, as Cushion already does, avoids the problem.
- Windows caveat: killing a parent does not kill its children.
  If a plugin's utility process spawns Sherpa and then crashes, Sherpa keeps running with a dead parent pid, and `taskkill /t` on the dead pid no longer finds it.
  This is the main argument for a core-owned process API (see the recommendation).

### 4. Other options considered and rejected

- `worker_threads` in main: separate event loop, same process.
  A native crash or OOM in a worker still takes down main, and the worker gets full Node.
- A hidden `BrowserWindow` with `nodeIntegration: true`: the older pattern for background Node work.
  It goes against items 2 to 4 of the [security checklist](https://github.com/electron/electron/blob/35-x-y/docs/tutorial/security.md), turns off the renderer sandbox ([sandbox.md](https://github.com/electron/electron/blob/35-x-y/docs/tutorial/sandbox.md)) and costs a full renderer.
  `utilityProcess` exists to replace it.
- Running plugin Node code in the renderer (Obsidian's model): would mean turning on `nodeIntegration` for the app window, which undoes the current hardening.

## Comparison

| | In main | `utilityProcess` | `child_process` |
|---|---|---|---|
| Plugin JS with Node APIs | Yes | Yes | Only via `ELECTRON_RUN_AS_NODE` |
| Sandbox | No | No | No |
| Plugin crash kills app | Native crash or OOM: yes | No | No |
| Plugin hang freezes app | Yes | No | No |
| Disable without restart | No | Yes, `kill()` | Yes |
| Direct channel to renderer | n/a | `MessagePort` | No |
| Per-plugin metrics | No | `app.getAppMetrics()` by `serviceName` | Manual |
| Extra processes | 0 | 1 per plugin, or 1 shared | 1 per binary |
| Wiring cost | Lowest | RPC over ports | Ad hoc per binary |

## How VS Code does it

- On desktop the extension host is a `utilityProcess` forked from main with `type: 'extensionHost'` ([extensionHostStarter.ts](https://github.com/microsoft/vscode/blob/main/src/vs/platform/extensions/electron-main/extensionHostStarter.ts), [utilityProcess.ts](https://github.com/microsoft/vscode/blob/main/src/vs/platform/utilityProcess/electron-main/utilityProcess.ts)).
  Microsoft contributed `utilityProcess` to Electron for this purpose; before that the host was a forked child process ([sandbox blog, Nov 2022](https://code.visualstudio.com/blogs/2022/11/28/vscode-sandbox)).
  It became the default in 1.75 ([release notes](https://code.visualstudio.com/updates/v1_75)), and the renderer sandbox shipped to everyone in 1.79 ([release notes](https://code.visualstudio.com/updates/v1_79)).
- All local extensions share one host process per window ([sandbox blog](https://code.visualstudio.com/blogs/2022/11/28/vscode-sandbox)).
  An experimental `extensions.experimental.affinity` setting can move chosen extensions into a separate host ([#164048](https://github.com/microsoft/vscode/issues/164048)).
  Host kinds are local (Node), web (WebWorker) and remote ([extension host docs](https://code.visualstudio.com/api/advanced-topics/extension-host)).
- Main creates a `MessageChannelMain` and hands one port to the host and one to the window, so the renderer and host talk directly "without impacting any other process, such as the main process" ([sandbox blog](https://code.visualstudio.com/blogs/2022/11/28/vscode-sandbox), [localProcessExtensionHost.ts](https://github.com/microsoft/vscode/blob/main/src/vs/workbench/services/extensions/electron-browser/localProcessExtensionHost.ts)).
  On top of the port runs `RPCProtocol`: proxy calls, request/reply/cancel/ack messages and an unresponsiveness monitor ([rpcProtocol.ts](https://github.com/microsoft/vscode/blob/main/src/vs/workbench/services/extensions/common/rpcProtocol.ts)).
- On a crash VS Code restarts the host automatically, up to 3 crashes in 5 minutes, then stops and offers "Restart Extension Host" and "Start Extension Bisect" ([abstractExtensionService.ts](https://github.com/microsoft/vscode/blob/main/src/vs/workbench/services/extensions/common/abstractExtensionService.ts), [nativeExtensionService.ts](https://github.com/microsoft/vscode/blob/main/src/vs/workbench/services/extensions/electron-browser/nativeExtensionService.ts)).
- Extensions "are free to spawn as many child processes as they require" ([sandbox blog](https://code.visualstudio.com/blogs/2022/11/28/vscode-sandbox)).
  "The extension host has the same permissions as VS Code itself"; the defence is a publisher-trust prompt at install time since 1.97 ([extension runtime security](https://code.visualstudio.com/docs/configure/extensions/extension-runtime-security)).
  Workspace Trust "can't prevent a malicious extension from executing code" ([workspace trust](https://code.visualstudio.com/docs/editing/workspaces/workspace-trust)).
- Native binaries ship in platform-specific packages built with `vsce package --target win32-x64`, and the docs name native Node modules as the main use case ([publishing](https://code.visualstudio.com/api/working-with-extensions/publishing-extension)).

## How Obsidian desktop does it

- Plugins `require('fs')` and `require('electron')` directly, and must bundle every other dependency into one `main.js` ([obsidian-api README](https://github.com/obsidianmd/obsidian-api/blob/master/README.md)).
  The sample plugin's esbuild config marks `electron` and Node built-ins as external ([esbuild.config.mjs](https://github.com/obsidianmd/obsidian-sample-plugin/blob/master/esbuild.config.mjs)).
  This implies plugins run in a renderer with Node integration; I found no official statement saying so.
- A plugin that uses Node or Electron APIs must set `isDesktopOnly: true` in its manifest ([manifest](https://docs.obsidian.md/Reference/Manifest), [submission requirements](https://docs.obsidian.md/Plugins/Releasing/Submission+requirements+for+plugins)).
- There is no process boundary, no sandbox and no permission system: "Obsidian cannot reliably restrict plugins to specific permissions or access levels", and plugins can "install additional programs" ([plugin security](https://obsidian.md/help/plugin-security)).
  Defences are Restricted Mode (on by default), automated malware scans with scorecards, and manual review of popular or flagged plugins.
- Developer policies ban obfuscation, telemetry and plugins that "Install or update themselves or their dependencies", and require the README to disclose network access and file access outside the vault ([developer policies](https://github.com/obsidianmd/obsidian-developer-docs/blob/main/en/Community%20directory/Developer%20policies.md)).
- No official doc covers native `.node` modules or `child_process`; both work in practice because plugins have Node, but the single-file `main.js` rule makes native modules awkward.

## Native modules and binaries on Windows x64

- Electron's ABI differs from Node's, so native modules must be rebuilt with `@electron/rebuild`, usually again after every Electron upgrade ([using native node modules](https://github.com/electron/electron/blob/35-x-y/docs/tutorial/using-native-node-modules.md)).
- Node-API is "ABI stable across versions of Node.js" ([n-api.md](https://github.com/nodejs/node/blob/v22.14.0/doc/api/n-api.md)), which is the only realistic way for a community plugin to survive Cushion's Electron upgrades.
  The Electron page does not mention Node-API, so treat "Node-API modules need no rebuild" as likely but untested here.
- Plain executables (Sherpa, Pandoc) have no ABI coupling at all, which is why the existing features already use them.

## Can the plugin process be locked down?

- Node 22's permission model (`--permission`, `--allow-fs-read`, `--allow-child-process`, ...) is stable since 22.13, but Node itself says it "does not protect against malicious code" and is not inherited by child processes ([permissions.md](https://github.com/nodejs/node/blob/v22.14.0/doc/api/permissions.md)).
- It is probably not usable in Electron anyway.
  Electron passes only an allowlist of Node CLI flags to Node, and `--permission` is not on it ([node_bindings.cc](https://github.com/electron/electron/blob/35-x-y/shell/common/node_bindings.cc)).
  Packaged apps also ignore almost all of `NODE_OPTIONS` ([environment-variables.md](https://github.com/electron/electron/blob/35-x-y/docs/api/environment-variables.md)).
  This is read from source, not tested.
- Conclusion: there is no cheap OS- or runtime-level sandbox for plugin Node code in Electron 35.

## Recommendation

1. A plugin manifest gets an optional Node entry, next to its renderer entry.
   Plugins without one never get a process.
2. The coordinator forks one `utilityProcess` per plugin with a Node entry, lazily on activation, with `serviceName` set to the plugin id.
   With only a handful of Node plugins expected (dictation, Pandoc export, NotebookLM), per-plugin processes cost little and give per-plugin crash attribution, metrics and kill-on-disable.
   If the measured memory cost is too high, fall back to VS Code's one shared host; the API does not change.
3. Two channels per plugin.
   Core services (workspace files, config, the process API below) are an RPC over `parentPort`, so main checks every call against the plugin's identity.
   Renderer-half to Node-half traffic goes over a brokered `MessagePort` that main never sees.
4. Core owns child processes.
   The Node half asks core to spawn its binaries (`process.spawn(binaryId, args)`), core spawns them with `execFile`/`spawn` and `shell: false`, tracks pids per plugin, and kills each tree with `taskkill /t` when the plugin stops or its host dies.
   Binaries are declared in the manifest and downloaded to `userData/plugins/<id>/bin`, never into the asar.
5. Copy VS Code's restart policy: restart a crashed plugin host automatically, give up after 3 crashes in 5 minutes, and tell the user which plugin failed.
6. The registry should only accept native code as plain executables or Node-API modules prebuilt for win32-x64.

Trade-offs of this recommendation:

- More wiring than loading into main: an RPC layer, a port relay in the preload, and a process supervisor.
  The supervisor mostly generalises what `sherpa-manager.ts` already does.
- One extra process per Node plugin.
- Plugins cannot share in-memory state with core; everything is async messages.
- None of it is a security boundary.

## Open risks

- Not a sandbox.
  A community plugin with a Node entry can do anything the user can; the registry review and trust model (#50) carry all of the security weight, as with VS Code and Obsidian.
- A plugin's Node half can still call `child_process` directly and bypass the core process API, so orphaned processes after a crash remain possible.
  Registry review has to enforce the API; Node gives no way to block it inside Electron.
- `utilityProcess.fork` from inside `app.asar` (for built-in plugins) and per-process memory on Windows are both untested.
- Node-API modules skipping `@electron/rebuild` is untested.
- The Node permission model in a utility process is untested; source reading says it will not work.
- The current `shell/exec` handler spawns through `cmd.exe` (`shell: true`) and its metacharacter filter does not include `%` or `^`.
  Moving NotebookLM to a plugin is a good moment to switch it to `shell: false` with resolved executable paths.
- Today any renderer code can call any `coordinator:*` method, so this design only helps once plugin renderer code stops getting the raw `electronAPI`.
