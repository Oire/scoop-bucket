# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is the **Oire Scoop bucket** — a Git repository of JSON manifests describing [Oire Software](https://oire.dev) apps for the [Scoop](https://scoop.sh) package manager on Windows.

A "bucket" in Scoop is just a public Git repo with a `bucket/` subdirectory containing one `.json` file per app. Users add it with `scoop bucket add oire https://github.com/Oire/scoop-bucket` and then install apps with `scoop install oire/<name>`.

There is no build step, no release pipeline for this repo itself — manifests are the product. The only automation needed is bumping `version`, `url`, and `hash` after upstream apps ship a new release.

## Repo layout

```
scoop-bucket/
├── bucket/             # manifests live here (one JSON file per app)
│   └── sic.json
├── README.md           # user-facing: how to add the bucket, what it contains
├── LICENSE
└── CLAUDE.md           # this file
```

## Naming conventions

- **Manifest filename == app name == install handle.** `bucket/sic.json` → `scoop install oire/sic`.
- **Lowercase, short, no dots, no publisher prefix.** Scoop is NOT winget — do not use `Oire.Sic` or similar. The bucket name is the namespace.
- Match the upstream repo name where reasonable (e.g., `github.com/Oire/sic` → `sic`).

## Authoring a new manifest

### 1. Scaffold (optional)

```bash
scoop create '<download-url>'
```

Interactive: asks you to pick the app name from URL segments. In non-interactive shells (including Claude Code's Bash tool) it errors but still writes a file named after the last URL segment. Rename it to `bucket/<app>.json` and rewrite the contents — the scaffold is barely more than an empty template.

Honestly, just copy an existing manifest and edit. Faster.

### 2. Get the SHA-256 from GitHub, not by downloading

```bash
gh release view v<VERSION> --repo Oire/<app> --json assets
```

Each asset object has a `digest` field formatted as `sha256:<hex>`. Paste the hex into the manifest's `hash`. **Do not** `curl` the ZIP and run `sha256sum` — the user has pushed back on this more than once. The API already has the value; use it.

### 3. Required fields

```json
{
  "version": "1.0.2.2",
  "description": "...",
  "homepage": "https://...",
  "license": "Apache-2.0",
  "architecture": {
    "64bit": {
      "url": "https://github.com/Oire/<app>/releases/download/v$version/<asset>.zip",
      "hash": "<sha256 hex, no prefix>"
    }
  },
  "bin": "<MainExe>.exe",
  "shortcuts": [["<MainExe>.exe", "<Friendly Name>"]],
  "checkver": { "github": "https://github.com/Oire/<app>" },
  "autoupdate": {
    "architecture": {
      "64bit": { "url": "https://github.com/Oire/<app>/releases/download/v$version/<asset>.zip" }
    }
  }
}
```

Notes:

- `license` uses [SPDX identifiers](https://spdx.org/licenses/): `Apache-2.0`, `MIT`, `GPL-3.0-or-later`, etc.
- `bin` can be a string (single exe) or an array (expose multiple shims).
- `shortcuts` creates Start Menu entries. Format: `[[exe, name], [exe2, name2, args]]`.
- Scoop accepts 4-part versions like `1.0.2.2` (unlike Chocolatey). GitVersion output works directly.
- `$version` in `autoupdate.url` is substituted by `checkver -Update`.

### 4. Validate locally

```bash
# Just check that checkver can parse the manifest and reach GitHub
powershell -NoProfile -Command "& ~/scoop/apps/scoop/current/bin/checkver.ps1 -App <name> -Dir bucket"

# Simulate an end-to-end install (will actually install the app!)
scoop bucket add oire-local C:/repos/Oire/scoop-bucket
scoop install oire-local/<name>
scoop uninstall oire-local/<name>
scoop bucket rm oire-local
```

## The portable-mode + `persist` pattern (important for Oire apps)

Oire desktop apps (starting with SIC!) detect **portable mode** by the presence of a `userdata/` folder next to the executable — see `src/Sic/Utils/Constants/App.cs`:

```csharp
public static readonly bool IsPortable = Directory.Exists(
    Path.Combine(AppContext.BaseDirectory, "userdata"));
```

When portable, the app reads/writes config, logs, and output under `<exe>/userdata/` instead of `%APPDATA%`. The portable ZIP distribution ships with an empty `userdata/` folder pre-created to opt into this mode.

**Under Scoop, this collides with the update model** — each version installs into `~/scoop/apps/<name>/<version>/`, and without intervention the `userdata/` folder would be orphaned on every update.

The fix is Scoop's `persist` directive:

```json
"persist": "userdata"
```

Scoop moves `userdata/` on first install to `~/scoop/persist/<name>/userdata/` and junctions it into every version folder. Updates leave it alone. `scoop uninstall` preserves it; `scoop uninstall -p` wipes it.

**When authoring a manifest for a new Oire app, check whether its portable ZIP ships a `userdata/` folder.** If yes, you almost certainly want `"persist": "userdata"`. If the app has other locally-written directories (e.g., `logs/`, `cache/`) that are NOT inside `userdata/`, add them to `persist` too:

```json
"persist": ["userdata", "logs"]
```

Document the persist behavior in the manifest's `notes` field so users know where their data lives.

## Release workflow (after upstream ships a new version)

```bash
# From this repo, with the new upstream release already published on GitHub:
powershell -NoProfile -Command "& ~/scoop/apps/scoop/current/bin/checkver.ps1 -App <name> -Dir bucket -Update"
```

`checkver -Update` will:

1. Detect the new version from GitHub releases.
2. Rewrite `version` and the `url` (from the `autoupdate` template).
3. Download the asset and recompute `hash`.
4. Leave a diff to review.

Review the diff, commit with a message like `sic: update to 1.0.3.0`, push. Scoop users get the update on their next `scoop update`.

### Batch-updating multiple apps

```bash
powershell -NoProfile -Command "& ~/scoop/apps/scoop/current/bin/checkver.ps1 * -Dir bucket -Update"
```

The `*` means "all manifests in the directory."

## Gotchas

- **ZIP with a top-level folder** (e.g., `sic-v1.0.2.2/Sic.exe` instead of `Sic.exe` at root) → add `"extract_dir": "sic-v$version"` or similar. SIC!'s portable ZIP intentionally has files at root (`7z a ... "$PortableTempDir\*"` in `installer/Build-Installer.ps1`), so no `extract_dir` needed.
- **Asset URL pattern changes between versions** → write an explicit `autoupdate.url` with `$version` substitutions, or provide a `regex` in `checkver` to extract the version differently.
- **Apps requiring elevation (drivers, services, shell extensions)** → bad fit for Scoop. Use Chocolatey for those; Scoop is strictly per-user.
- **Pre-release/beta channels** → either ship a separate manifest (`sic-beta.json`) or use Scoop's `suggest` field. Do not mix stable and beta versions in one manifest.
- **The `/dev/null` rule** (from user memory): never redirect output to null devices in bash commands in this repo either. Capture to a variable or just let it print.

## Related channels (not managed here)

- **winget**: manifests live in `microsoft/winget-pkgs`. Updated from the main SIC! repo via `wingetcreate` — see [SIC!'s CLAUDE.md](https://github.com/Oire/sic/blob/master/CLAUDE.md) for the command (note the `|x64` URL suffix quirk).
- **Chocolatey**: not yet published. Would live in a separate repo or a `choco/` folder in the main app repo, not here. Chocolatey uses the Inno Setup installer (not the portable ZIP) and has its own moderation queue.

Scoop is the only channel this repo manages.
