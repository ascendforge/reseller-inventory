# Reseller Inventory

Desktop inventory tracker for eBay and Facebook Marketplace reselling. Built with Tauri 2 — vanilla HTML/JS frontend, no bundler, no framework.

## Features

- Auto-generated item numbers (ITM-0001, ITM-0002, ...)
- Tracks cost, asking price, sold price, shipping charged to buyer, shipping paid, and platform fees
- Per-item and total profit calculation
- Status flow: In stock → Listed → Sold (with undo)
- Search, status filter, CSV export
- **Local storage**: data lives in a plain JSON file at
  `~/Library/Application Support/com.argonexus.reseller-inventory/inventory.json`
  — easy to back up, no cloud, no account.

## Option A — Build on the Mac (one-time setup, ~10 min)

Run these in Terminal on the Mac:

```bash
# 1. Xcode command line tools (skip if already installed)
xcode-select --install

# 2. Rust
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
source "$HOME/.cargo/env"

# 3. Node (if not installed): https://nodejs.org or `brew install node`

# 4. From this project folder:
npm install
npm run tauri build
```

The app lands at:
`src-tauri/target/release/bundle/macos/Reseller Inventory.app` (drag to Applications)
and a `.dmg` in `src-tauri/target/release/bundle/dmg/`.

For development with hot reload: `npm run tauri dev`

> **First launch note:** the app is unsigned, so macOS may block it. Right-click the app → Open → Open (only needed once). Or: System Settings → Privacy & Security → "Open Anyway".

## Option B — Build with GitHub Actions (no Mac dev setup)

1. Push this folder to a GitHub repo
2. Go to the repo's **Actions** tab → **Build macOS app** → **Run workflow**
3. Download the `.dmg` from the workflow artifacts (~5 min build)

The workflow builds a universal binary (Apple Silicon + Intel).

## Quick test without building

`src/index.html` works standalone in any browser (falls back to browser localStorage). Handy for trying UI changes before a full build.

## Data backup

Everything is in one file:

```
~/Library/Application Support/com.argonexus.reseller-inventory/inventory.json
```

Copy it anywhere to back it up; replace it to restore.
