# Cushion

A local-first markdown workspace with an integrated AI agent.
Cushion keeps a small core for writing and chatting, and everything else ships as plugins.

## Language

### Plugins

**Core**:
The features that ship inside Cushion itself and are not plugins.
_Avoid_: base app, built-ins

**Plugin**:
An installable unit of functionality that extends Cushion through the plugin API.
_Avoid_: extension, add-on, module

**Built-in plugin**:
A plugin that ships with the Cushion installer and can be disabled but not uninstalled.
_Avoid_: bundled extension, bundled plugin, core plugin

**Optional plugin**:
A plugin that is not in the installer and is installed from the registry through the marketplace.
_Avoid_: community plugin, external extension, third-party plugin

**Pack**:
A named group of plugins that installs and uninstalls together.
_Avoid_: bundle, suite, collection

### Distribution

**Registry**:
The list of plugins and packs that Cushion can install from outside the installer. Only the maintainer adds entries.
_Avoid_: store, catalog, index

**Marketplace**:
The UI for browsing the registry and installing from it.
_Avoid_: store, plugin browser
