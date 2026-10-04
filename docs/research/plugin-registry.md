# How Obsidian's community registry and GitHub Releases distribution work

Research for [#53](https://github.com/Aleexc12/cushion/issues/53), part of the plugin system map [#48](https://github.com/Aleexc12/cushion/issues/48).
Checked on 2026-10-05.
Terms follow `CONTEXT.md`, so "registry" means the curated list and "marketplace" means the browsing UI.

## Short answer

Obsidian ran a zero-cost registry for six years.
A JSON file in a public GitHub repo listed every plugin, authors added themselves by pull request, a GitHub Action linted the PR, and a human reviewed it.
The app downloaded the list from GitHub, read each plugin's `manifest.json` from the plugin repo's default branch, and fetched `main.js`, `manifest.json` and `styles.css` from the GitHub Release whose tag equals that version.
Only the first version was reviewed.
Every later release reached users as soon as the author bumped `manifest.json`, with no checksum and no signature.

On 2026-05-12 Obsidian replaced the PR flow with a hosted service, the Community directory at `community.obsidian.md`.
Submissions now go through a web form, every release is scanned automatically, and manual review only covers popular, featured and flagged plugins.
The GitHub repo still exists, but a bot overwrites its JSON hourly from the directory and pull requests are disabled.
The listed plugin count went from 3,640 just before the switch to 8,405 today, which shows how far behind the human review queue had fallen.

For Cushion the PR-era design is a good fit, because the registry is small, curated, and must cost nothing.
Two parts of it should not be copied.
Cushion should pin each version and its SHA-256 in the registry instead of trusting "whatever the plugin repo's HEAD says", and clients should read one prebuilt index instead of calling GitHub once per plugin.

## Two eras

| | PR era (2020 to May 2026) | Community directory (since 2026-05-12) |
|---|---|---|
| Source of truth | `community-plugins.json` in `obsidianmd/obsidian-releases` | Obsidian's own service at `community.obsidian.md` |
| Submission | PR appending one entry to the JSON | Web form, signed in with an Obsidian account plus a linked GitHub account |
| Automated checks | GitHub Action on the PR, metadata only | Scanner on every release, covering manifest, release assets, source code and build verification |
| Human review | Every new plugin, once | Popular, featured and flagged plugins |
| Updates | Author bumps `manifest.json`, nothing reviewed | Each release scanned, failing plugins removed from search within 24 hours |
| Hosting cost | Zero, all GitHub | Obsidian runs a service behind Cloudflare |
| GitHub repo role | Registry | Read-only mirror, PRs disabled |

Sources are the commit that removed the PR templates and validation workflows ([d4f0694](https://github.com/obsidianmd/obsidian-releases/commit/d4f0694), 2026-05-15), the mirror workflow ([mirror-community-json.yml](https://github.com/obsidianmd/obsidian-releases/blob/master/.github/workflows/mirror-community-json.yml)), and the launch post ([The future of plugins](https://obsidian.md/blog/future-of-plugins/), 2026-05-12).
`gh pr view` against the repo now returns "Pull requests are disabled for this repository".

## Repository layout

`obsidianmd/obsidian-releases` holds no app source code.
It hosts desktop releases and the community lists ([README](https://github.com/obsidianmd/obsidian-releases/blob/master/README.md)).

| File | Size | Content |
|---|---:|---|
| `community-plugins.json` | 2.5 MB (651 KB gzipped) | Array of 8,405 plugin entries |
| `community-plugins-removed.json` | 20 KB | 175 removed plugins with a reason |
| `community-plugin-deprecation.json` | 1 KB | Map of plugin id to versions that must not be used |
| `community-plugin-stats.json` | 2.5 MB | Download counts per plugin and per version |
| `community-css-themes.json` | 158 KB | Theme list |
| `community-css-themes-removed.json` | 3 KB | Removed themes |
| `desktop-releases.json` | 1 KB | App auto-update pointer with hash and signature |
| `.github/workflows/mirror-community-json.yml` | | Hourly mirror from the directory |
| `.github/workflows/plugin-stat.yml` | | Daily stats mirror from the directory |

Sizes and counts come from the `master` branch on 2026-10-05.

## Index formats

### `community-plugins.json`

Each entry has exactly five string fields, and the validation workflow rejected any other key.

```json
{
  "id": "obsidian-git",
  "name": "Git",
  "author": "Vinzent",
  "description": "Integrate Git version control with automatic backup and other advanced features.",
  "repo": "vinzent03/obsidian-git"
}
```

There is no version, no asset URL, no checksum and no platform field.
The entry only points to a repo, and everything else is resolved live from that repo.
The mirror workflow now enforces the same shape plus `owner/name` repo syntax and unique ids.

### `community-plugins-removed.json`

```json
{ "id": "arrows", "name": "Arrows", "reason": "Archived repository" }
```

The old validator refused new plugins reusing a removed id, so a removed id can't be taken over by a new author.

### `community-plugin-deprecation.json`

```json
{ "templater-obsidian": ["0.5.2", "0.5.3"] }
```

This is a per-version blocklist, the only version-level control the PR-era registry had.

### `community-plugin-stats.json`

```json
{ "13th-age-statblocks": { "downloads": 5298, "updated": 1709754303000, "0.1.0": 76, "0.1.1": 79 } }
```

Until June 2026 a nightly Action built this file by paging `GET /repos/{owner}/{repo}/releases` for every plugin with a personal access token, and counting the `download_count` of each release's `manifest.json` asset ([old plugin-stat.yml](https://github.com/obsidianmd/obsidian-releases/blob/c884be96dd43af0bafa7c1a1d80e767d03f2b6ec/.github/workflows/plugin-stat.yml)).
GitHub's download counter was the only install metric, at zero cost.

### `desktop-releases.json`

```json
{
  "latestVersion": "1.13.7",
  "downloadUrl": "https://github.com/obsidianmd/obsidian-releases/releases/download/v1.13.7/obsidian-1.13.7.asar.gz",
  "hash": "aSU+OaoLmA48+W6eiopL7Wtkge9wIc12L2eHJmLY0lo=",
  "signature": "1uwLwiTp...Hg=="
}
```

Obsidian's own app updates carry a base64 SHA-256 and a 256-byte signature.
Plugins get neither.

## Submission flow in the PR era

1. The author published a public GitHub repo with `README.md`, `LICENSE` and `manifest.json` at the root.
2. The author created a GitHub Release whose tag equals `manifest.json`'s `version` exactly, with no `v` prefix, and attached `main.js`, `manifest.json` and optionally `styles.css` as individual assets.
3. The author forked `obsidian-releases`, appended one entry to the end of `community-plugins.json`, and opened a PR using the plugin template.
4. A `pull_request_target` workflow ([validate-plugin-entry.yml](https://github.com/obsidianmd/obsidian-releases/blob/c884be96dd43af0bafa7c1a1d80e767d03f2b6ec/.github/workflows/validate-plugin-entry.yml)) commented with errors and warnings, labeled the PR, and assigned `ObsidianReviewBot` once it passed.
5. A human from the Obsidian team reviewed the code and merged the PR, or labeled it `Changes requested`.
6. A stale bot closed PRs labeled `Validation failed` after 14 days ([f6e829b](https://github.com/obsidianmd/obsidian-releases/commit/f6e829b)).

The PR template ([plugin.md](https://github.com/obsidianmd/obsidian-releases/blob/c884be96dd43af0bafa7c1a1d80e767d03f2b6ec/.github/PULL_REQUEST_TEMPLATE/plugin.md)) was a checklist with a maintenance pledge, platforms tested, release assets present, tag equals version, id matches, README, policies read, license and attribution.
The validator rejected PRs whose body lacked any checklist line.

The validation bot checked these things.

- The PR changes only `community-plugins.json`.
- The entry has exactly `id`, `name`, `description`, `author`, `repo`.
- The PR author owns the repo or is a public member of the owning org, which blocks submitting someone else's code.
- The id matches `^[a-z0-9-_]+$`, contains no "obsidian" and doesn't end in "plugin".
- The name avoids "Obsidian" and "Plugin".
- The description is at most 250 characters, ends in `.?!)`, and avoids "Obsidian" and "This plugin".
- The id, name and repo are unique, and the id was never used by a removed plugin.
- `manifest.json` exists at the repo root, has the required keys and no unknown keys, and its id, name and description equal the PR entry.
- `version` matches `^[0-9.]+$`.
- A release tagged `version` exists and has `main.js` and `manifest.json` assets.
- GitHub detects a license on the repo.
- Issues are enabled, as a warning only.

All of this used the workflow's `GITHUB_TOKEN`, so the checks cost nothing.
Labels like `Ready for review`, `Changes requested`, `Additional review required`, `Installation not recommended` and `Skipped code scan` drove the human queue.

## Submission flow in the Community directory era

1. The author signs in to `community.obsidian.md` with an Obsidian account and connects GitHub, which proves repo ownership ([Set up and claim](https://github.com/obsidianmd/obsidian-developer-docs/blob/main/en/Community%20directory/Set%20up%20and%20claim.md)).
2. The author submits the repo URL through a form and agrees to the developer policies.
3. The directory reads `manifest.json` at HEAD of the default branch and the release whose tag matches ([Submit your plugin](https://github.com/obsidianmd/obsidian-developer-docs/blob/main/en/Plugins/Releasing/Submit%20your%20plugin.md)).
4. After each release the scanner checks the manifest, release assets and source code, runs the first of the `build`, `build:plugin` or `compile` scripts, and verifies that the build output matches the release assets ([Manage your plugin or theme](https://github.com/obsidianmd/obsidian-developer-docs/blob/main/en/Community%20directory/Manage%20your%20plugin%20or%20theme.md), [FAQ](https://github.com/obsidianmd/obsidian-developer-docs/blob/main/en/Community%20directory/Frequently%20asked%20questions.md)).
5. A plugin "won't be installable from within Obsidian until any errors from the automated review are resolved", which implies the app now consults the directory before installing.
6. The directory polls for new releases periodically, and the author can force a check.

The listing scorecard reports "verified GitHub artifact attestations", Obsidian API usage such as Vault Read or Vault Write, and disclosures like clipboard access or external domains ([Community directory help](https://github.com/obsidianmd/obsidian-help/blob/master/en/Extending%20Obsidian/Community%20directory.md)).
The official release workflow now includes an `actions/attest` step for `main.js`, `manifest.json` and `styles.css`, described as "recommended" ([Release your plugin with GitHub Actions](https://github.com/obsidianmd/obsidian-developer-docs/blob/main/en/Plugins/Releasing/Release%20your%20plugin%20with%20GitHub%20Actions.md)).
The launch post says manual reviews "will continue" but shift toward "popular plugins, featured plugins, and issues flagged by the community", and that the new system cleared "over 2,300 queued submissions in the last few days".

## How the app resolves a plugin to release assets

From the README section "How community plugins are pulled", which still describes the GitHub-based flow:

1. The app reads `community-plugins.json`, and searches on `name`, `author` and `description`.
2. Opening a plugin's detail page fetches `manifest.json` and `README.md` from the plugin repo.
3. The repo's `manifest.json` only decides the latest version.
4. If `minAppVersion` is newer than the running app, the app reads `versions.json` from the repo root, a map of plugin version to minimum app version, and picks the newest compatible plugin version ([Versions](https://github.com/obsidianmd/obsidian-developer-docs/blob/main/en/Reference/Versions.md)).
5. Install looks for a GitHub Release tagged exactly with that version and downloads `manifest.json`, `main.js` and `styles.css` into `<vault>/.obsidian/plugins/<id>/`.

Release asset URLs follow `https://github.com/{owner}/{repo}/releases/download/{tag}/{name}`.
That URL answers `302` to a signed, short-lived `release-assets.githubusercontent.com` URL, observed with `curl -I` on 2026-10-05.
These are web downloads, not REST API calls, so they don't consume the 60 requests per hour REST quota.

Obsidian's app is closed source.
The exact hosts the current app uses (raw GitHub, the GitHub mirror, or `community.obsidian.md/assets/community-plugins.json`) can't be confirmed from primary sources.
The hourly mirror and its 95% retention guard suggest older clients still read the GitHub copy.

## Update checks

- Plugins never update automatically, which the help docs frame as a security choice ([Community plugins](https://github.com/obsidianmd/obsidian-help/blob/master/en/Extending%20Obsidian/Community%20plugins.md)).
- The user presses "Check for updates", and the app compares each installed plugin's version with the latest version from the plugin repo's `manifest.json`.
- "Update all" or per-plugin "Update" downloads the release assets again.
- The deprecation file can block specific versions.
- In the PR era nobody reviewed a new version before it reached users who pressed Update.
  Pushing a new `manifest.json` and a matching release was enough.
  The directory now scans each release, but a failing version still sits on GitHub until the scan runs.

## GitHub limits for unauthenticated clients

| Surface | Limit | Source |
|---|---|---|
| REST API, unauthenticated | 60 requests per hour per IP | [REST rate limits](https://docs.github.com/en/rest/using-the-rest-api/rate-limits-for-the-rest-api) |
| REST API, authenticated user | 5,000 per hour | same |
| `GITHUB_TOKEN` in Actions | 1,000 per hour per repo | same |
| GitHub App installation | 5,000 to 12,500 per hour | same |
| Secondary limits | 100 concurrent, 900 points per minute, 80 content-creating requests per minute | same |
| Conditional requests | A `304` is free only when sent with an `Authorization` header, so unauthenticated `304`s still count | [REST best practices](https://docs.github.com/en/rest/using-the-rest-api/best-practices-for-using-the-rest-api) |
| `raw.githubusercontent.com` | Rate limited for unauthenticated users since May 2025, number not published | [GitHub changelog 2025-05-08](https://github.blog/changelog/2025-05-08-updated-rate-limits-for-unauthenticated-requests/) |
| Raw caching | Fastly, `Cache-Control: max-age=300`, strong `ETag`, `Access-Control-Allow-Origin: *` | Observed headers on 2026-10-05 |
| Release assets | Under 2 GiB per file, up to 1,000 assets per release, no total size or bandwidth limit | [About releases](https://docs.github.com/en/repositories/releasing-projects-on-github/about-releases) |
| GitHub Pages | 1 GB site, 100 GB per month soft bandwidth, `429` when throttled | [Pages limits](https://docs.github.com/en/pages/getting-started-with-github-pages/github-pages-limits) |

The practical consequence is that a client must never call the REST API per plugin.
One index request plus direct release downloads stays far from every limit.
Any flow that reads `manifest.json` or `versions.json` from each plugin repo scales with installed plugin count, and the raw limit is undocumented, so a heavy user can hit `429`s with no way to predict when.

## Integrity options

The PR-era Obsidian registry had none for plugins.
Options available on GitHub at zero cost, from cheapest to strongest:

1. **SHA-256 pinned in the registry.**
   The REST API now returns `"digest": "sha256:..."` on every release asset, observed on `vinzent03/obsidian-git` 2.41.1.
   A registry CI job can copy that digest into the index at review time, and the client verifies the download before extracting.
   This catches a swapped asset, but only as far as the index itself can be trusted.
2. **Immutable releases.**
   Once enabled on a repo, the tag can't move and assets can't be modified or deleted while the release exists, and GitHub generates a release attestation ([Immutable releases](https://docs.github.com/en/code-security/supply-chain-security/understanding-your-software-supply-chain/immutable-releases)).
   The release API exposes an `immutable` boolean, so the registry CI can require it.
   `gh release verify` and `gh release verify-asset` check it ([Verifying a release](https://docs.github.com/en/code-security/supply-chain-security/understanding-your-software-supply-chain/verifying-the-integrity-of-a-release)).
3. **Artifact attestations.**
   `actions/attest` signs build provenance through Sigstore's public-good instance for public repos, with a transparency log entry, and reaches SLSA v1.0 Build Level 2 ([Artifact attestations](https://docs.github.com/en/actions/concepts/security/artifact-attestations)).
   It proves which workflow, commit and repo built an asset.
   Verifying in the client needs a Sigstore verifier, and the documented attestation list endpoint needs a token ([Attestations API](https://docs.github.com/en/rest/users/attestations)), so this belongs in registry CI rather than in the app.
4. **A signed index.**
   The maintainer signs the index with an offline key and the app ships the public key, the same pattern Obsidian uses for `desktop-releases.json`.
   This is the only option that survives a compromised GitHub account on the registry repo.

## What breaks at scale

- **Human review.**
  The queue grew until 2,300 or more submissions were waiting, and the switch to automated review more than doubled the listed count in under five months.
  A single maintainer hits this ceiling much earlier.
- **Review once, trust forever.**
  Only the first version was checked.
  An author, or anyone who takes over the author's GitHub account, could ship arbitrary code to every installed user on the next update.
  This was the main reason for the per-release scanner.
- **No pinned versions.**
  "Latest" is whatever the plugin repo's HEAD `manifest.json` says, so the registry can't hold back a bad release except through the deprecation blocklist.
- **One JSON file for everything.**
  The list reached 2.5 MB, and every client downloads it whole to search.
  The validator found the new entry by taking the last array element, which forced append-only PRs and merge conflicts between concurrent submissions.
- **Per-plugin GitHub calls.**
  Detail pages and update checks fetch `manifest.json`, `README.md` and `versions.json` from each plugin repo, against an unpublished raw limit.
- **Stats.**
  Paging every plugin's releases nightly with one personal token stops fitting at 5,000 requests per hour once the plugin count or release history gets large.
- **Repo renames and transfers.**
  The entry stores `owner/repo`, so moving a repo needs a registry edit, and admins handle it by hand.

## Alternatives worth comparing

| Registry | Index | Artifacts | Review | Cost to run |
|---|---|---|---|---|
| Obsidian PR era | One JSON in a GitHub repo | Plugin's own GitHub Releases | Human, first version only | Zero |
| Obsidian Community directory | Hosted service, mirrored to GitHub | Plugin's own GitHub Releases | Automated scan of every release, human for some | Own service |
| [Logseq marketplace](https://github.com/logseq/marketplace) | One `packages/<id>/manifest.json` per plugin in a GitHub repo | A zip attached to the plugin's GitHub Release | Human, by PR | Zero |
| [Zed extensions](https://zed.dev/docs/extensions/publishing/publishing-guide) | `extensions.toml` with a pinned version, plus a git submodule per extension | Zed CI builds and publishes to Zed's own registry | Human, every PR, including updates | Zed hosts |
| [winget-pkgs](https://learn.microsoft.com/en-us/windows/package-manager/package/manifest) | YAML manifests per version in a GitHub repo, compiled into an index served from `cdn.winget.microsoft.com/cache` | Publisher's own URL, often GitHub Releases, with required `InstallerSha256` | A PR per version | Microsoft CDN |

Logseq's one-file-per-plugin layout avoids the append conflict.
Zed and winget pin a version per entry, so every update is a new PR.
Winget pins a SHA-256 for every installer and serves a prebuilt index from a CDN instead of letting clients walk the repo.
BRAT is a community plugin that installs straight from any GitHub repo's releases, and Obsidian points beta testers at it ([Beta-testing plugins](https://github.com/obsidianmd/obsidian-developer-docs/blob/main/en/Plugins/Releasing/Beta-testing%20plugins.md)).
Cushion could treat that pattern as its developer sideloading path.

## Implications for Cushion

These follow from the map's constraints of zero hosting cost, maintainer review of every submission, Windows x64 only, and plugins that may ship native binaries.

- A GitHub repo with PR review is enough for the registry.
  The PR era worked for six years and only broke at a review volume Cushion won't see soon.
- Pin versions in the registry, as Zed and winget do, so updates go through review too.
  The repo-HEAD approach contradicts "the maintainer reviews every submission".
  A bot that opens an update PR when a plugin publishes a release keeps the per-update work small.
- Store the SHA-256 of every asset in the registry entry, copied from the release API's `digest` at review time, and verify it in the app before extracting.
- Have registry CI build one generated index with id, version, asset URL, digest, size, platform and minimum Cushion version.
  The app then fetches a single file with `ETag` caching, and downloads artifacts from `github.com/.../releases/download/...`, which uses no REST quota.
- Use one file per plugin in the registry repo, as Logseq does, so concurrent PRs don't conflict.
- Native binaries don't fit Obsidian's three-file format, and Obsidian's policy forbids plugins that "install or update themselves or their dependencies".
  Cushion needs a single zip asset per version, named for the platform such as `win32-x64`, with the binary inside and its digest pinned.
  The 2 GiB per-asset limit leaves plenty of room.
- Require immutable releases on plugin repos, which registry CI can check through the `immutable` field.
  Verify artifact attestations in CI as well, where a token is available.
- Signing the generated index with a maintainer key is optional and protects against a compromised registry repo.
  Obsidian already does this for its own app updates.

## Unknowns

- Which host the current Obsidian app reads the plugin list from.
  The app is closed source and the docs don't say.
- The numeric rate limit for unauthenticated `raw.githubusercontent.com`.
  GitHub announced the limit in May 2025 without a number.
- Whether a native binary inside a plugin zip triggers SmartScreen or antivirus warnings on Windows when Cushion extracts and runs it.
  This needs a prototype, and it matters for the trust model ticket.

## Sources

- [obsidianmd/obsidian-releases README](https://github.com/obsidianmd/obsidian-releases/blob/master/README.md)
- [mirror-community-json.yml](https://github.com/obsidianmd/obsidian-releases/blob/master/.github/workflows/mirror-community-json.yml) and [plugin-stat.yml](https://github.com/obsidianmd/obsidian-releases/blob/master/.github/workflows/plugin-stat.yml)
- [Historical validate-plugin-entry.yml](https://github.com/obsidianmd/obsidian-releases/blob/c884be96dd43af0bafa7c1a1d80e767d03f2b6ec/.github/workflows/validate-plugin-entry.yml), [plugin PR template](https://github.com/obsidianmd/obsidian-releases/blob/c884be96dd43af0bafa7c1a1d80e767d03f2b6ec/.github/PULL_REQUEST_TEMPLATE/plugin.md) and [old stats workflow](https://github.com/obsidianmd/obsidian-releases/blob/c884be96dd43af0bafa7c1a1d80e767d03f2b6ec/.github/workflows/plugin-stat.yml), at the last commit before removal
- [Commit d4f0694 removing PR templates and validation](https://github.com/obsidianmd/obsidian-releases/commit/d4f0694)
- [The future of plugins, Obsidian blog, 2026-05-12](https://obsidian.md/blog/future-of-plugins/)
- Obsidian developer docs source in [obsidianmd/obsidian-developer-docs](https://github.com/obsidianmd/obsidian-developer-docs), covering Submit your plugin, Submission requirements, Developer policies, Set up and claim, Manage your plugin or theme, FAQ, Manifest, Versions, Release with GitHub Actions and Beta-testing
- Obsidian help docs source in [obsidianmd/obsidian-help](https://github.com/obsidianmd/obsidian-help), covering Community plugins, Plugin security and Community directory
- GitHub docs on [REST rate limits](https://docs.github.com/en/rest/using-the-rest-api/rate-limits-for-the-rest-api), [REST best practices](https://docs.github.com/en/rest/using-the-rest-api/best-practices-for-using-the-rest-api), [releases](https://docs.github.com/en/repositories/releasing-projects-on-github/about-releases), [immutable releases](https://docs.github.com/en/code-security/supply-chain-security/understanding-your-software-supply-chain/immutable-releases), [release verification](https://docs.github.com/en/code-security/supply-chain-security/understanding-your-software-supply-chain/verifying-the-integrity-of-a-release), [artifact attestations](https://docs.github.com/en/actions/concepts/security/artifact-attestations), [attestations API](https://docs.github.com/en/rest/users/attestations) and [Pages limits](https://docs.github.com/en/pages/getting-started-with-github-pages/github-pages-limits)
- [GitHub changelog, updated rate limits for unauthenticated requests](https://github.blog/changelog/2025-05-08-updated-rate-limits-for-unauthenticated-requests/)
- [Logseq marketplace README](https://github.com/logseq/marketplace), [Zed extensions repo](https://github.com/zed-industries/extensions) and [publishing guide](https://zed.dev/docs/extensions/publishing/publishing-guide), [winget manifest docs](https://learn.microsoft.com/en-us/windows/package-manager/package/manifest)
- Direct observation with `curl -I` and `gh api` on 2026-10-05 for headers, redirects, asset digests and file sizes
