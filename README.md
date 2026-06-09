# Reseller Inventory

Desktop inventory tracker for eBay and Facebook Marketplace reselling. Built with Tauri 2 — vanilla HTML/JS frontend, no bundler, no framework. Runs on **Windows and macOS**, with built-in auto-update from GitHub Releases.

## Features

- Auto-generated item numbers (ITM-0001, ITM-0002, ...)
- Tracks cost, asking price, sold price, shipping charged to buyer, shipping paid, and platform fees
- Per-item and total profit calculation
- Status flow: In stock → Listed → Sold (with undo)
- Search, status filter, and **CSV export** (native "Save As" dialog — choose where the file goes)
- **Auto-update**: checks GitHub Releases on launch (and via a "Check for updates" link); installs and restarts on your confirmation
- **Local storage**: data lives in a plain JSON file (see [Where your data lives](#where-your-data-lives)) — easy to back up, no cloud, no account

## Installing (for users)

1. Go to the [**Releases**](https://github.com/ascendforge/reseller-inventory/releases) page.
2. Download the installer for your OS:
   - **Windows**: the `.exe` setup file
   - **macOS**: the `.dmg`
3. Run it. The app is **not code-signed**, so you'll see a one-time "unknown publisher" warning:
   - **Windows**: SmartScreen → **More info** → **Run anyway**
   - **macOS**: right-click the app → **Open** → **Open** (or System Settings → Privacy & Security → **Open Anyway**)

Updates after that are automatic: when a newer release is published, the app offers to install it on next launch.

## Where your data lives

One JSON file, written automatically on every change:

- **Windows**: `%APPDATA%\com.argonexus.reseller-inventory\inventory.json`
  (i.e. `C:\Users\<you>\AppData\Roaming\com.argonexus.reseller-inventory\inventory.json`)
- **macOS**: `~/Library/Application Support/com.argonexus.reseller-inventory/inventory.json`

Copy it anywhere to back it up; replace it to restore. You can also use **Export CSV** in the app for a portable copy.

## Releasing a new version (for the maintainer)

Releases are built and published automatically by GitHub Actions when you push a version tag.

1. Bump `version` in `src-tauri/tauri.conf.json` (and keep `package.json` / `src-tauri/Cargo.toml` in sync).
2. Commit the bump.
3. Tag and push:
   ```bash
   git tag v1.2.3
   git push origin v1.2.3
   ```
4. The [Release workflow](.github/workflows/release.yml) builds a macOS universal binary and a Windows installer, signs the updater artifacts, and publishes a GitHub Release containing the installers, their `.sig` signatures, and `latest.json` (the updater manifest the app reads).

Installed apps detect the new version on their next launch (or via **Check for updates**) and offer to install it.

### Update signing

Updates are verified against an embedded public key (in `tauri.conf.json` under `plugins.updater.pubkey`). The matching private key is stored as the GitHub Actions secret `TAURI_SIGNING_PRIVATE_KEY`. **If you lose that private key you can't publish working updates** — back it up somewhere safe.

## Local development

```bash
npm install
npm run tauri dev    # hot-reloading desktop window
```

Building locally (optional — CI handles release builds):

```bash
npm run tauri build
```

Requires [Rust](https://www.rust-lang.org/tools/install) and [Node](https://nodejs.org). On macOS you also need Xcode command line tools (`xcode-select --install`).

## Quick UI tweaks without building

`src/index.html` works standalone in any browser — it falls back to browser `localStorage` (and a normal download for CSV export) when not running inside Tauri. Handy for trying UI changes before a full build. Note this browser data is separate from the desktop app's JSON file.
