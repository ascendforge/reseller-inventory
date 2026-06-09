# CSV Export + GitHub Auto-Updater Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make CSV export work reliably in the packaged desktop app, and add a GitHub-backed cross-platform (Windows + macOS) auto-updater with a CI release pipeline.

**Architecture:** Stays a Tauri 2 desktop app with a single-file vanilla HTML/JS frontend (`src/index.html`, `withGlobalTauri: true`, no bundler). New capability comes from official Tauri v2 plugins (`dialog`, `fs`, `updater`, `process`) plus config; distribution moves to a tag-triggered `tauri-action` workflow that publishes signed installers and a `latest.json` updater manifest to public GitHub Releases.

**Tech Stack:** Tauri 2, Rust, vanilla JS, GitHub Actions, `tauri-apps/tauri-action`, `tauri-plugin-{store,dialog,fs,updater,process}`.

> **Note on testing:** This project has no test framework, and adding one for a single HTML file + config is out of scope (YAGNI). "Verification" here means building/running the real app and observing behavior, inspecting generated files, and checking CI output — appropriate to this project's nature. Each task ends with a concrete observable check and a commit.

> **Note on the frontend API:** With `withGlobalTauri: true`, every plugin is exposed under `window.__TAURI__.<plugin>` (the existing code already uses `window.__TAURI__.store`). So: `window.__TAURI__.dialog`, `.fs`, `.updater`, `.process`. No `import` statements — this is a no-bundler single file.

---

## File structure

| File | Responsibility | Change |
| --- | --- | --- |
| `src-tauri/Cargo.toml` | Rust deps | Add `dialog`, `fs`, `updater`, `process` plugins |
| `src-tauri/src/lib.rs` | Plugin registration | Register the four new plugins (updater/process desktop-only) |
| `src-tauri/tauri.conf.json` | App + bundle + updater config | Icon, bundle targets, `createUpdaterArtifacts`, `plugins.updater` |
| `src-tauri/capabilities/default.json` | Window permissions | Add dialog/fs/updater/process permissions |
| `src/index.html` | Frontend logic + UI | Rewrite `exportCsv`; add updater prompt, flow, and header button |
| `.github/workflows/release.yml` | CI release pipeline | New; replaces `build-mac.yml` |
| `README.md` | Docs | Update build/release/update instructions |

---

## Task 1: Bootstrap the public GitHub repo

**Files:** none (git + GitHub operations only)

The local folder is now a git repo (the spec commit `6fb741c` exists) but has no remote. Create the public repo under the `ascendforge` account and push everything. `node_modules/`, `src-tauri/target/`, `src-tauri/gen/` are already gitignored.

- [ ] **Step 1: Stage and commit the full project**

```bash
cd "C:/dev/personal/reseller-inventory"
git add -A
git -c user.name="Matt Kynaston" -c user.email="m.holloway.kynaston@gmail.com" \
  commit -m "Initial project import"
```

Expected: a commit containing `src/`, `src-tauri/`, `.github/`, `README.md`, `app-icon.png`, etc. (not `target`/`node_modules`).

- [ ] **Step 2: Confirm the right GitHub account is active**

```bash
gh auth switch --user ascendforge
gh auth status
```

Expected: `Active account: true` under `ascendforge`.

- [ ] **Step 3: Create the public repo and push**

```bash
gh repo create ascendforge/reseller-inventory --public --source=. --remote=origin --push
```

Expected: repo created at https://github.com/ascendforge/reseller-inventory and `main` pushed.

- [ ] **Step 4: Verify**

```bash
gh repo view ascendforge/reseller-inventory --json visibility,url --jq '.visibility + " " + .url'
```

Expected: `PUBLIC https://github.com/ascendforge/reseller-inventory`.

---

## Task 2: Fix bundle config (Windows icon + cross-platform targets)

**Files:**
- Modify: `src-tauri/tauri.conf.json` (the `bundle` block)

`icons/icon.ico` already exists on disk but isn't referenced; Windows packaging needs it. `targets: "all"` lets each OS build its applicable bundles (NSIS/MSI on Windows, app/dmg on macOS) without cross-platform target errors.

- [ ] **Step 1: Update the bundle block**

Replace the existing `bundle` object in `src-tauri/tauri.conf.json` with:

```json
  "bundle": {
    "active": true,
    "targets": "all",
    "createUpdaterArtifacts": true,
    "icon": [
      "icons/32x32.png",
      "icons/128x128.png",
      "icons/128x128@2x.png",
      "icons/icon.icns",
      "icons/icon.ico"
    ]
  }
```

(`createUpdaterArtifacts: true` is required in Tauri v2 to emit the update artifacts and `.sig` files; it's harmless until the updater plugin is added in Task 5.)

- [ ] **Step 2: Verify the dev build still launches**

```bash
cd "C:/dev/personal/reseller-inventory"
npm run tauri dev
```

Expected: compiles and the "Reseller Inventory" window opens with no icon/bundle error. Close the window (or Ctrl+C) after confirming.

- [ ] **Step 3: Commit**

```bash
git add src-tauri/tauri.conf.json
git commit -m "build: wire icon.ico into bundle, target all platforms, enable updater artifacts"
```

---

## Task 3: CSV export — verify current behavior, then fix with native Save As

**Files:**
- Modify: `src-tauri/Cargo.toml` (deps)
- Modify: `src-tauri/src/lib.rs` (register dialog + fs)
- Modify: `src-tauri/capabilities/default.json` (permissions)
- Modify: `src/index.html` (`exportCsv`)

- [ ] **Step 1: Verify whether the current export already works in the packaged app**

```bash
cd "C:/dev/personal/reseller-inventory"
npm run tauri dev
```

In the window: add one item, click **Export CSV**. Observe whether a `.csv` file is actually saved anywhere (check Downloads, or whether a save dialog appears).

Expected (hypothesis): no file is saved / no dialog appears, because the `<a download>` data-URL trick does not trigger a file write in WebView2. **If a file IS correctly saved, skip Steps 2–6** and instead just add a code comment noting it works, then commit. Otherwise continue.

- [ ] **Step 2: Add the dialog and fs plugin dependencies**

In `src-tauri/Cargo.toml`, under `[dependencies]` (after `tauri-plugin-store = "2"`), add:

```toml
tauri-plugin-dialog = "2"
tauri-plugin-fs = "2"
```

- [ ] **Step 3: Register the plugins**

In `src-tauri/src/lib.rs`, change the builder so it reads:

```rust
#[cfg_attr(mobile, tauri::mobile_entry_point)]
pub fn run() {
    tauri::Builder::default()
        .plugin(tauri_plugin_store::Builder::new().build())
        .plugin(tauri_plugin_dialog::init())
        .plugin(tauri_plugin_fs::init())
        .run(tauri::generate_context!())
        .expect("error while running tauri application");
}
```

- [ ] **Step 4: Grant the permissions**

In `src-tauri/capabilities/default.json`, change the `permissions` array to:

```json
  "permissions": [
    "core:default",
    "store:default",
    "dialog:default",
    "fs:allow-write-text-file"
  ]
```

No fs path-scope is needed: the dialog plugin's `save` command grants write access to the exact file the user selects at runtime.

- [ ] **Step 5: Rewrite `exportCsv` in `src/index.html`**

Replace the entire `exportCsv` function with this async version (Tauri branch + browser fallback). The CSV-building lines are unchanged:

```javascript
async function exportCsv() {
  const head = ['Item Number','Name','Status','Platform','Date Added','Date Sold','Cost','Asking Price','Sold Price','Shipping Charged','Shipping Paid','Fees','Profit'];
  const rows = state.items.map(it => [
    it.number, '"' + it.name.replace(/"/g, '""') + '"', it.status, it.platform,
    it.added || '', it.soldDate || '',
    (it.cost||0).toFixed(2), (it.askPrice||0).toFixed(2),
    (it.soldPrice||0).toFixed(2), (it.shipCharged||0).toFixed(2),
    (it.shipCost||0).toFixed(2), (it.fees||0).toFixed(2),
    it.status === 'sold' ? profitOf(it).toFixed(2) : ''
  ].join(','));
  const csv = [head.join(','), ...rows].join('\n');
  const filename = 'inventory-' + new Date().toISOString().slice(0,10) + '.csv';

  // Desktop (Tauri): native Save As dialog + write chosen file
  if (window.__TAURI__ && window.__TAURI__.dialog && window.__TAURI__.fs) {
    try {
      const path = await window.__TAURI__.dialog.save({
        defaultPath: filename,
        filters: [{ name: 'CSV', extensions: ['csv'] }],
      });
      if (!path) return; // user cancelled
      await window.__TAURI__.fs.writeTextFile(path, csv);
      toast('CSV exported');
    } catch (e) {
      toast('Could not export — try again');
      console.error(e);
    }
    return;
  }

  // Browser fallback (standalone index.html)
  const a = document.createElement('a');
  a.href = 'data:text/csv;charset=utf-8,' + encodeURIComponent(csv);
  a.download = filename;
  document.body.appendChild(a);
  a.click();
  a.remove();
  toast('CSV exported');
}
```

- [ ] **Step 6: Verify the native export**

```bash
npm run tauri dev
```

In the window: add an item, click **Export CSV** → a native Save As dialog appears → choose a location → confirm the `.csv` file exists at that path with a header row and the item row. Then open `src/index.html` directly in a browser and confirm Export CSV still downloads a file (fallback path).

- [ ] **Step 7: Commit**

```bash
git add src-tauri/Cargo.toml src-tauri/src/lib.rs src-tauri/capabilities/default.json src/index.html
git commit -m "feat: native Save As for CSV export in desktop app, keep browser fallback"
```

---

## Task 4: Add updater + process Rust plugins

**Files:**
- Modify: `src-tauri/Cargo.toml`
- Modify: `src-tauri/src/lib.rs`

- [ ] **Step 1: Add the dependencies (updater/process are desktop-only)**

In `src-tauri/Cargo.toml`, after the `[dependencies]` block, add a desktop-target section:

```toml
[target."cfg(not(any(target_os = \"android\", target_os = \"ios\")))".dependencies]
tauri-plugin-updater = "2"
tauri-plugin-process = "2"
```

- [ ] **Step 2: Register them (desktop-only) in `src-tauri/src/lib.rs`**

Update `run()` so it conditionally registers the desktop plugins:

```rust
#[cfg_attr(mobile, tauri::mobile_entry_point)]
pub fn run() {
    let builder = tauri::Builder::default()
        .plugin(tauri_plugin_store::Builder::new().build())
        .plugin(tauri_plugin_dialog::init())
        .plugin(tauri_plugin_fs::init());

    #[cfg(desktop)]
    let builder = builder
        .plugin(tauri_plugin_updater::Builder::new().build())
        .plugin(tauri_plugin_process::init());

    builder
        .run(tauri::generate_context!())
        .expect("error while running tauri application");
}
```

- [ ] **Step 3: Verify it compiles**

```bash
cd "C:/dev/personal/reseller-inventory/src-tauri"
cargo check
```

Expected: `Finished` with no errors (the updater will warn at runtime that no `pubkey`/`endpoints` are configured until Task 5 — that's fine for `cargo check`).

- [ ] **Step 4: Commit**

```bash
cd "C:/dev/personal/reseller-inventory"
git add src-tauri/Cargo.toml src-tauri/src/lib.rs
git commit -m "feat: add updater and process plugins (desktop)"
```

---

## Task 5: Generate signing key, configure updater, set CI secrets

**Files:**
- Modify: `src-tauri/tauri.conf.json` (add `plugins.updater`)
- Modify: `src-tauri/capabilities/default.json` (add updater/process permissions)

- [ ] **Step 1: Generate the updater signing keypair**

```bash
cd "C:/dev/personal/reseller-inventory"
npx tauri signer generate -w "$HOME/.tauri/reseller-inventory.key"
```

When prompted for a password, set one (remember it). This writes the **private** key to `~/.tauri/reseller-inventory.key` (NEVER commit this) and prints the **public** key, plus writes `~/.tauri/reseller-inventory.key.pub`.

- [ ] **Step 2: Read the public key**

```bash
cat "$HOME/.tauri/reseller-inventory.key.pub"
```

Copy the full single-line public key string (starts with `dW50cnVzdGVk...` base64).

- [ ] **Step 3: Add the `plugins` block to `src-tauri/tauri.conf.json`**

Add a top-level `plugins` key (sibling of `app` and `bundle`). Paste the public key from Step 2 in place of `<PUBLIC_KEY>`:

```json
  "plugins": {
    "updater": {
      "endpoints": [
        "https://github.com/ascendforge/reseller-inventory/releases/latest/download/latest.json"
      ],
      "pubkey": "<PUBLIC_KEY>"
    }
  }
```

- [ ] **Step 4: Add updater + process permissions to `src-tauri/capabilities/default.json`**

Update the `permissions` array to its final form:

```json
  "permissions": [
    "core:default",
    "store:default",
    "dialog:default",
    "fs:allow-write-text-file",
    "updater:default",
    "process:allow-restart"
  ]
```

- [ ] **Step 5: Verify it still compiles/runs**

```bash
npm run tauri dev
```

Expected: window opens; no updater config error in the console (it will fail the actual *check* with a network/404 until a release exists — acceptable now).

- [ ] **Step 6: Set the GitHub Actions secrets (private key + password)**

```bash
gh secret set TAURI_SIGNING_PRIVATE_KEY < "$HOME/.tauri/reseller-inventory.key" \
  --repo ascendforge/reseller-inventory
gh secret set TAURI_SIGNING_PRIVATE_KEY_PASSWORD \
  --body "<the password from Step 1>" --repo ascendforge/reseller-inventory
```

- [ ] **Step 7: Verify secrets exist**

```bash
gh secret list --repo ascendforge/reseller-inventory
```

Expected: both `TAURI_SIGNING_PRIVATE_KEY` and `TAURI_SIGNING_PRIVATE_KEY_PASSWORD` listed.

- [ ] **Step 8: Commit (config only — no keys)**

```bash
git add src-tauri/tauri.conf.json src-tauri/capabilities/default.json
git commit -m "feat: configure updater endpoint, pubkey, and permissions"
```

---

## Task 6: Updater frontend — launch check, prompt, install, manual button

**Files:**
- Modify: `src/index.html` (header markup, an update overlay, JS functions, init hook)

- [ ] **Step 1: Add a "Check for updates" link to the header**

In `src/index.html`, replace the `<header>...</header>` block (lines ~109–112) with:

```html
<header>
  <h1><span class="dot"></span>Inventory</h1>
  <span class="sub">eBay &amp; Facebook Marketplace
    <a href="#" id="checkUpdateLink" onclick="checkForUpdates({silent:false}); return false;" style="margin-left:12px; font-size:12px;">Check for updates</a>
  </span>
</header>
```

- [ ] **Step 2: Add the update overlay markup**

Immediately after the existing sell overlay (the `<div class="overlay" id="sellOverlay">...</div>` block ending around line 205), add:

```html
<div class="overlay" id="updateOverlay">
  <div class="dialog">
    <h3>Update available</h3>
    <div class="dnum" id="updateMsg"></div>
    <div class="form-actions">
      <button class="btn-primary" id="updateInstallBtn" onclick="installUpdate()">Install &amp; restart</button>
      <button class="btn-ghost" onclick="closeUpdate()">Later</button>
    </div>
  </div>
</div>
```

(Reuses the existing `.overlay` / `.dialog` / `.form-actions` / `.btn-primary` / `.btn-ghost` styles — identical to `#sellOverlay`. The overlay is shown/hidden via the `open` class: `.overlay.open { display: flex }`, exactly as `openSell`/`closeDialog` do.)

- [ ] **Step 3: Add the updater JS**

In the `<script>` section, add these functions and a module-level variable (near the other `let` declarations add `let pendingUpdate = null;`):

```javascript
async function checkForUpdates({ silent = false } = {}) {
  if (!(window.__TAURI__ && window.__TAURI__.updater)) {
    if (!silent) toast('Updates are only available in the desktop app');
    return;
  }
  try {
    const update = await window.__TAURI__.updater.check();
    if (!update) {
      if (!silent) toast("You're on the latest version");
      return;
    }
    pendingUpdate = update;
    document.getElementById('updateMsg').textContent =
      'Version ' + update.version + ' is available. Install now? The app will restart.';
    document.getElementById('updateOverlay').classList.add('open');
  } catch (e) {
    if (!silent) toast('Could not check for updates');
    console.error(e);
  }
}

function closeUpdate() {
  document.getElementById('updateOverlay').classList.remove('open');
}

async function installUpdate() {
  if (!pendingUpdate) return;
  const btn = document.getElementById('updateInstallBtn');
  btn.disabled = true;
  btn.textContent = 'Downloading…';
  try {
    await pendingUpdate.downloadAndInstall((event) => {
      if (event.event === 'Progress') btn.textContent = 'Downloading…';
      if (event.event === 'Finished') btn.textContent = 'Installing…';
    });
    await window.__TAURI__.process.relaunch();
  } catch (e) {
    btn.disabled = false;
    btn.textContent = 'Install & restart';
    toast('Update failed — try again');
    console.error(e);
  }
}
```

(The overlay is shown by adding the `open` class and hidden by removing it — identical to how `openSell`/`closeDialog` toggle `#sellOverlay`.)

- [ ] **Step 4: Trigger a silent check on launch**

In `src/index.html`, update the `init()` IIFE (lines ~433–436) to:

```javascript
(async function init() {
  try { await initStorage(); await loadData(); } catch (e) { console.error(e); }
  render();
  checkForUpdates({ silent: true });
})();
```

- [ ] **Step 5: Verify the UI wiring (no release yet)**

```bash
npm run tauri dev
```

Expected: window opens; clicking **Check for updates** in the header shows a toast "You're on the latest version" (or "Could not check…" if no release exists yet — both confirm the wiring and that no exception is thrown). The silent launch check must not show any error popup. Close when done.

- [ ] **Step 6: Commit**

```bash
git add src/index.html
git commit -m "feat: in-app update check on launch + manual button with install/restart flow"
```

---

## Task 7: CI release workflow

**Files:**
- Delete: `.github/workflows/build-mac.yml`
- Create: `.github/workflows/release.yml`

- [ ] **Step 1: Remove the old macOS-only workflow**

```bash
git rm .github/workflows/build-mac.yml
```

- [ ] **Step 2: Create `.github/workflows/release.yml`**

```yaml
name: Release

on:
  push:
    tags:
      - 'v*'

jobs:
  release:
    permissions:
      contents: write
    strategy:
      fail-fast: false
      matrix:
        include:
          - platform: 'macos-latest'
            args: '--target universal-apple-darwin'
          - platform: 'windows-latest'
            args: ''
    runs-on: ${{ matrix.platform }}
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node
        uses: actions/setup-node@v4
        with:
          node-version: 20

      - name: Setup Rust
        uses: dtolnay/rust-toolchain@stable
        with:
          targets: ${{ matrix.platform == 'macos-latest' && 'aarch64-apple-darwin,x86_64-apple-darwin' || '' }}

      - name: Install frontend dependencies
        run: npm install

      - uses: tauri-apps/tauri-action@v0
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          TAURI_SIGNING_PRIVATE_KEY: ${{ secrets.TAURI_SIGNING_PRIVATE_KEY }}
          TAURI_SIGNING_PRIVATE_KEY_PASSWORD: ${{ secrets.TAURI_SIGNING_PRIVATE_KEY_PASSWORD }}
        with:
          tagName: ${{ github.ref_name }}
          releaseName: 'Reseller Inventory ${{ github.ref_name }}'
          releaseBody: 'See the assets below to download and install.'
          releaseDraft: false
          prerelease: false
          uploadUpdaterJson: true
          updaterJsonPreferNsis: true
          args: ${{ matrix.args }}
```

- [ ] **Step 3: Commit and push**

```bash
git add .github/workflows/release.yml
git commit -m "ci: cross-platform tag-triggered release workflow with updater manifest"
git push origin main
```

- [ ] **Step 4: Verify the workflow is registered**

```bash
gh workflow list --repo ascendforge/reseller-inventory
```

Expected: `Release` appears in the list.

---

## Task 8: End-to-end release + update verification

**Files:**
- Modify: `src-tauri/tauri.conf.json` (version bump, second release)
- Modify: `README.md` (docs)

- [ ] **Step 1: Cut the first release (v1.0.0)**

The current version is `1.0.0`. Tag and push:

```bash
git tag v1.0.0
git push origin v1.0.0
```

- [ ] **Step 2: Watch the build**

```bash
gh run watch --repo ascendforge/reseller-inventory
```

Expected: both matrix jobs (macOS, Windows) succeed. On failure, read logs with `gh run view --log-failed` and fix before continuing.

- [ ] **Step 3: Verify the release assets**

```bash
gh release view v1.0.0 --repo ascendforge/reseller-inventory --json assets --jq '.assets[].name'
```

Expected: a Windows NSIS setup `.exe`, a macOS `.dmg` and `.app.tar.gz`, their `.sig` files, and `latest.json`.

- [ ] **Step 4: Install v1.0.0 locally**

Download the Windows NSIS installer from the release, run it (click through the SmartScreen "unknown publisher" warning), and launch the installed app. Confirm **Check for updates** reports "You're on the latest version."

- [ ] **Step 5: Bump to v1.0.1**

In `src-tauri/tauri.conf.json`, change `"version": "1.0.0"` to `"version": "1.0.1"`. Also bump `version` in `package.json` and `src-tauri/Cargo.toml` to `1.0.1` to keep them aligned.

```bash
git add src-tauri/tauri.conf.json package.json src-tauri/Cargo.toml
git commit -m "chore: bump version to 1.0.1"
git push origin main
git tag v1.0.1
git push origin v1.0.1
```

- [ ] **Step 6: Verify the auto-update end-to-end**

After the v1.0.1 build finishes and publishes, launch the **installed v1.0.0** app. Expected: on launch (or via the manual button) the "Update available — Version 1.0.1" overlay appears; clicking **Install & restart** downloads, installs, and relaunches into 1.0.1. Confirm the version is now 1.0.1 (via the manual check reporting up-to-date and the data still intact in `inventory.json`).

- [ ] **Step 7: Update the README**

In `README.md`, replace the macOS-only build instructions with: the cross-platform release flow (tag `vX.Y.Z` → CI publishes), how end users install (download from Releases, click through the unsigned warning), how auto-update works (checks on launch + manual button), and note the Windows data path `%APPDATA%\com.argonexus.reseller-inventory\inventory.json` alongside the macOS path.

- [ ] **Step 8: Commit**

```bash
git add README.md
git commit -m "docs: cross-platform build, release, and auto-update instructions"
git push origin main
```

---

## Self-review notes

- **Spec coverage:** Local save unchanged (no task needed — confirmed in Task 8 Step 6 that data persists across update). CSV export → Task 3. Updater runtime → Tasks 4–6. Signing key → Task 5. CI → Task 7. Repo bootstrap → Task 1. icon.ico prerequisite → Task 2. Release flow → Task 8. All spec sections mapped.
- **Type/name consistency:** `pendingUpdate`, `checkForUpdates`, `installUpdate`, `closeUpdate`, `#updateOverlay`, `#updateMsg`, `#updateInstallBtn` are defined together in Task 6 and used consistently. `exportCsv` signature stays `exportCsv()` (called from existing `onclick="exportCsv()"`, which tolerates the now-async function).
- **Known adaptation:** verification is manual (run/observe) because the project has no test harness and adding one is out of scope.
- **Verified against the source:** the update overlay reuses `#sellOverlay`'s exact markup classes (`.overlay`/`.dialog`/`.form-actions`/`.btn-primary`/`.btn-ghost`) and show mechanism (`.open` class), confirmed in `src/index.html`.
- **One gated assumption:** that the current `<a download>` export is in fact broken in WebView2 (Task 3 Step 1 gates whether the fix is applied).
