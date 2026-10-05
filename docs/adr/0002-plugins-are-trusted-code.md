# Plugins are trusted code

A plugin can do anything Cushion itself can do.
Its renderer code shares the app's JavaScript realm and its optional Node entry runs with full Node in a `utilityProcess`, so no permission could actually be enforced.
Security rests on authorship instead: only the maintainer publishes to the registry, and Cushion installs a plugin only if its bytes match the SHA-256 pinned in the registry index.
The plugin context object is the supported API contract, not a security boundary.

This replaces the "Scoped API surface" claim in the old extension system docs, which described a restriction the code never enforced.

## Considered options

- Permissions declared in the manifest and shown at install, but not enforced. Rejected because it promises protection that does not exist.
- Enforced permissions through per-plugin iframes or out-of-renderer execution. Rejected because plugins render React into core-owned containers and extend the editor, and the catalog is first-party.

## Consequences

- Built-in and optional plugins use one public API with no privilege tiers. Unstable additions are marked experimental, which exempts them from compatibility guarantees but not from use by any plugin.
- Local plugins load only with Developer mode on, after a one-time confirmation per folder that states the code gets full access. They are labelled "Local" and never auto-update.
- A workspace never supplies or installs plugin code. Its config only enables or disables plugins that are already installed.
- The renderer is hardened against the content it shows, not against plugins: a strict CSP, `sandbox: true`, no navigation, one outbound link path for http(s) and `mailto:`, no generic command execution, `workspace/open` limited to paths main already knows, sender-frame checks on every IPC handler, and a typed preload API.
- The registry index is not separately signed while the installer itself ships unsigned from GitHub Releases. Revisit when the app gets code signing or auto-update.
- Third-party plugins need a new trust model, not an extension of this one.
