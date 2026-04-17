# Oire Scoop bucket

A [Scoop](https://scoop.sh) bucket for [Oire Software](https://oire.dev) apps.

## Add the bucket

```powershell
scoop bucket add oire https://github.com/Oire/scoop-bucket
```

## Available apps

| App | Description |
|---|---|
| [sic](bucket/sic.json) | SIC! (Simple Image Converter) — accessible image format converter (JPG, PNG, WEBP, ICO, BMP, TIFF, GIF, AVIF) with optional resizing. |

## Install an app

```powershell
scoop install oire/sic
```

## Notes on persisted data

Apps that carry user data (config, logs, output) use Scoop's `persist` directive so that data survives updates. For SIC!, that data lives under `~/scoop/persist/sic/userdata/`. To also wipe it on uninstall, pass `-p`:

```powershell
scoop uninstall -p sic
```
