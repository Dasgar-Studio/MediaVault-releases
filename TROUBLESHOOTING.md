# Troubleshooting & Error Handling

This guide covers common issues, built-in recovery mechanisms, and error handling behaviors in **MediaVault** across storage, encryption, plugins, updates, and user interface states.

---

## Quick Diagnosis

| Symptom | Cause | Fast Fix |
| :--- | :--- | :--- |
| Cannot create a new vault (`FOLDER_EXISTS`) | A folder with the same name (case-insensitive) already exists at the destination | Pick a different vault name or open the existing folder with **Open Library** |
| External edits do not show up right away | `400ms` write-stability check + `300ms` watcher debounce | Wait ~1 second after saving externally or switch sections to rescan |
| Restored item appeared in `Misc/General` | Its original section or category folder was deleted while the item was in Trash | Move or edit the item's section/category from its Detail sheet |
| Forgotten Private Vault PIN | PIN cannot be recovered directly due to zero-knowledge encryption | Click **Forgot PIN?** and enter your `VAULT-XXXX-XXXX-XXXX-XXXX` recovery key |
| Plugin `.zip` fails to install | Missing `plugin.json`, missing entry script, or nested folder depth > 1 | Ensure `plugin.json` and `main.js` sit at the archive root or 1 folder deep |
| No update prompt on startup | Offline/timeout (`6000ms`), dev mode, or Microsoft Store managed build | Check manually in **Settings → About** or via the **Microsoft Store** app |

---

## 1. Vault & Markdown File Issues

### 1.1. Vault Name Collision (`FOLDER_EXISTS`)
* **Behavior:** When creating a new vault (`vault:initialize-new`), MediaVault checks case-insensitively whether a directory with the same name already exists in the chosen parent path (`FOLDER_EXISTS`) to prevent overwriting existing data.
* **Resolution:** Choose a unique vault name, or click **Open Library** if you intended to mount an existing vault folder.

### 1.2. External Edits Not Appearing Immediately
* **Behavior:** MediaVault monitors your vault using a background filesystem watcher (`chokidar`) with a `400ms` write-stability delay (`awaitWriteFinish`) and a `300ms` burst debounce so it never reads half-written files from external editors like Obsidian or VS Code.
* **Resolution:** Wait 1 second after saving externally. If a cloud sync client (OneDrive, Dropbox, Google Drive) locked the file during sync, switch sections or restart MediaVault to trigger a fresh incremental scan.

### 1.3. Corrupted or Malformed YAML Frontmatter
* **Behavior:** If an `.md` file contains invalid YAML syntax, `gray-matter` may fail to parse its metadata and log an error while skipping the broken file so the rest of your library still loads.
* **Resolution:**
  1. Open the affected `.md` file in any text editor.
  2. Verify that the top `---` and bottom `---` delimiters are intact and that values containing colons (`:`) or special characters are wrapped in quotes (`"Title: Subtitle"`).
  3. Inside MediaVault, undefined properties are stripped automatically before saving so YAML serialization never breaks.

### 1.4. Missing Cover Thumbnails
* **Behavior:** Cover images are stored locally inside `<VaultRoot>/.assets/` (or legacy `IMGs/`) and resolved via the `vault-asset://` protocol or Base64 cache.
* **Resolution:**
  * Confirm the image uses a supported extension (`.jpg`, `.jpeg`, `.png`, `.webp`, `.gif`, `.svg`).
  * If a remote URL failed to download due to network restrictions or hotlink protection, download the image manually and attach it via the local file picker or **Vault Gallery Picker**.

### 1.5. Accidentally Deleted Items or Sections
* **Deleted Items:** Open **Trash** from the sidebar to restore soft-deleted `.md` files back to their original directory.
* **Deleted Parent Folder:** If the original section or category directory no longer exists on disk, restoring an item from Trash automatically routes it to `<VaultRoot>/Misc/General/` and updates its `type` and `category` frontmatter so no data is lost.

---

## 2. Encrypted Vault (`.secure_vault/`) Errors

### 2.1. Wrong PIN / Authentication Failure
* **Behavior:** MediaVault secures `.secure_vault/manifest.enc` with **AES-256-GCM** using `100,000` PBKDF2-SHA256 iterations. Entering an incorrect PIN fails GCM authentication tag verification immediately (`Invalid PIN or corrupted key`) without exposing partial data.
* **Resolution:** Re-enter your 4-to-6-digit PIN carefully.

### 2.2. Forgotten PIN (Offline Recovery)
* **Resolution:**
  1. Click **Forgot PIN?** on the PIN pad modal.
  2. Enter or paste your one-time recovery key (`VAULT-XXXX-XXXX-XXXX-XXXX`) generated during initial setup.
  3. Set a new 4-to-6-digit PIN. MediaVault unwraps and re-wraps your master key in memory without re-encrypting `.secure_vault/manifest.enc`.
* **Warning:** Because MediaVault is strictly zero-knowledge, if both your PIN and your `VAULT-XXXX-...` recovery key are lost, `.secure_vault/manifest.enc` **cannot** be decrypted.

### 2.3. Automatic Session Lock
* **Behavior:** Active sessions automatically lock after **10 minutes** of inactivity (`600s`) or immediately when pressing `Ctrl + L`, zero-filling (`0x00`) the 32-byte master key buffer in volatile RAM.
* **Resolution:** Re-enter your PIN to start a new unlocked session.

---

## 3. Plugin Installation & Runtime Errors

### 3.1. Invalid Plugin Package (`.zip`)
* **Behavior:** When installing a local or remote `.zip` archive, MediaVault extracts it to a temporary OS folder and verifies `plugin.json` (either at the archive root or inside a single top-level folder, ignoring `__MACOSX` and hidden dotfiles), required manifest fields (`id`, `name`, `version`), and the entry script (`main.js` by default).
* **Resolution:**
  * Ensure `plugin.json` is not nested two or more folders deep.
  * Confirm the entry file specified in `manifest.entry` or `manifest.main` exists inside the archive.
  * Malformed `plugin.json` files inside `<VaultRoot>/.plugins/` are logged via `console.warn` and skipped during `scanPlugins` so they never crash or block valid plugins.

### 3.2. Remote Plugin Download Failures
* **Behavior:** Remote installations validate `http://` / `https://` URLs and follow up to **5 HTTP redirects**. If the server returns a non-`200` status or exceeds 5 redirects, installation aborts cleanly and cleans up temporary files.
* **Resolution:** Verify your internet connection or download the `.zip` manually and drag it onto the **Plugins** dropzone.

### 3.3. Plugin Fails to Render or Throws Runtime Errors
* **Resolution:**
  1. Open **Plugins** and toggle the problematic plugin off to run `deactivatePlugin` (which cleans up injected `<style>` tags, custom languages, and UI action hooks).
  2. Click **Open Folder** to inspect or remove the plugin directory inside `<VaultRoot>/.plugins/`.

---

## 4. Application Updates & UI Recovery

### 4.1. Startup Update Check Timeout (MediaVault Edge / GitHub Releases)
* **Behavior:** On GitHub Releases builds, MediaVault checks for updates once `1200ms` after launch with a strict **`6000ms` timeout**. If you are offline or GitHub does not respond in time, the check cancels silently without showing error popups.
* **Resolution:** Check manually later from **Settings → About** when online.

### 4.2. Microsoft Store Builds (MediaVault Atlas)
* **Behavior:** Store builds (`process.windowsStore === true`) disable the built-in GitHub OTA updater (`status: 'store-managed'`).
* **Resolution:** Manage updates directly through the **Microsoft Store** app or **Windows Update**.

### 4.3. Unsaved Changes Guard & UI Crash Recovery
* **Navigation Guard:** Leaving **Settings** or **Edit Details** with unsaved edits triggers a confirmation dialog so changes are not discarded accidentally.
* **React Error Boundary (`AppErrorBoundary`):** If an unexpected rendering error occurs, MediaVault catches the crash at the root boundary and displays a one-click **Reload** screen without touching your vault files on disk.