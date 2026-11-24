# MediaVault Plugin Developer Specification & API Reference

MediaVault incorporates an extensible, sandboxed plugin architecture designed to let third-party developers build full-page tooling, custom views, catalog import/export pipelines, metadata scrapers, custom UI action buttons, runtime localization packs, and automated asset downloaders.

This document details the plugin runtime lifecycle, manifest schema, execution environment, `pluginApi` surface, filesystem layout, IPC bridge methods, and bundling pipeline.

---

## 1. Directory & Filesystem Architecture

All plugins reside inside an isolated `.plugins/` directory located at the root of the active vault:

    <VaultPath>/
    └── .plugins/                 # Plugin storage directory
        └── <plugin-folder>/      # Unique folder per plugin
            ├── plugin.json       # Required plugin manifest
            ├── main.js           # Distributed ESM bundle evaluated by the app
            └── styles.css        # Scoped stylesheet (optional, auto-injected)

> **Important Bundling Rule:** MediaVault loads `main.js` into memory via `Blob URL` (`import(blobUrl)`), which does not resolve relative `./` module imports or external `.json` files at runtime. Multi-file source codebases (`src/`) must be bundled into a single self-contained `main.js` before packaging.

### Installing & Accessing Plugins
- **Local Offline ZIP Installation:** Users can drag and drop any `.zip` plugin package directly onto the **Plugin Library (`PluginsView`)** dropzone or select it via the file picker (`window.api.selectPluginZip` / `window.api.installPluginZip`).
- **On-Demand Official Catalog:** Users can browse and install verified extensions from the **Official Plugin Catalog (`ExploreView`)**, which displays a first-visit local execution notice and fetches `registry.json` on demand via `window.api.installRemotePlugin`.
- **Opening the Plugins Directory:** Users and developers can open `.plugins/` in their native OS file explorer via `window.api.openPluginsFolder(vaultBasePath)`. MediaVault guarantees directory presence by recursively running `fs.mkdir(pluginsDir, { recursive: true })` prior to launching the system shell.

---

## 2. Plugin Manifest (`plugin.json`)

Every plugin directory must contain a valid `plugin.json` in its root:

    {
        "id": "list-transfer",
        "name": "List Transfer",
        "version": "1.1.0",
        "author": "Alejandro Dastard",
        "description": "Bulk import and export library items via CSV or raw text.",
        "main": "main.js",
        "styles": "styles.css",
        "slots": ["pluginsView"],
        "permissions": ["vault:write", "network:download"]
    }

### Manifest Schema Reference

| Property | Type | Required | Description |
| :--- | :--- | :--- | :--- |
| `id` | `string` | **Yes** | Unique identifier (kebab-case). Used for slot resolution (`${id}-view`) and CSS scoping. |
| `name` | `string` | **Yes** | Human-readable name displayed in the Plugin Library manager. |
| `version` | `string` | **Yes** | Semantic versioning format (`MAJOR.MINOR.PATCH`). |
| `author` | `string` | No | Author, organization, or maintainer name. |
| `description` | `string` | No | Short functional overview. Displayed in plugin cards. |
| `icon` | `string` | No | Lucide icon name displayed when no custom cover is shown (defaults to `"Puzzle"`). |
| `main` | `string` | No | Entry JavaScript bundle filename (also accepts `entry`; defaults to `main.js`). |
| `styles` | `string` | No | Stylesheet filename (auto-detected if `styles.css` is present). |
| `slots` | `string[]` | No | Declared UI injection targets (`"pluginsView"`, `"toolbar"`, `"detail"`, `"floating"`). |
| `permissions` | `string[]` | No | Declared capabilities (`"vault:read"`, `"vault:write"`, `"network:download"`). |

---

## 3. Dynamic Runtime & Lifecycle Execution

### Module Sandbox & Evaluation
MediaVault evaluates plugin source code without external runtime dependencies:

1. **Source Loading:** `window.api.loadPluginSource({ vaultBasePath, folderName, file })` reads the file buffer over IPC.
2. **ESM In-Memory Instantiation:** The raw JS string is converted to a dynamic blob:

       const blob = new Blob([jsSource], { type: 'application/javascript' });
       const blobUrl = URL.createObjectURL(blob);
       const module = await import(/* @vite-ignore */ blobUrl);
       URL.revokeObjectURL(blobUrl);

3. **Activation Hook:** If exported, `await module.activate(pluginApi)` executes immediately.
4. **Style Injection:** If `styles.css` exists, its content is read via `window.api.loadPluginStyles` and mounted into the application `<head>` with the tag ID `plugin-style-${plugin.id}`.
5. **Automatic Teardown:** When disabled or uninstalled, MediaVault calls `module.deactivate()`, removes all registered floating/toolbar/detail buttons, unregisters any plugin-owned custom languages, and removes `<style id="plugin-style-${plugin.id}">`.

### Lifecycle Methods (`main.js` Exports)
A plugin module must export standard ESM lifecycle functions:

    // Global activation handler triggered when the plugin is toggled ON
    export async function activate(api) {
        api.registerSlotView(`${api.pluginId}-view`, (container) => {
            mount(container, api);
        });
    }

    // Global deactivation handler triggered when toggled OFF or during vault unmount
    export async function deactivate() {
        // Tear down intervals, global listeners, or network long-polls
    }

    // UI Mount lifecycle invoked when navigating to full-screen tool routes
    export function mount(container, api) {
        container.innerHTML = `<div class="plugin-root">...</div>`;

        return {
            unmount() {
                container.innerHTML = '';
            }
        };
    }

    // UI Unmount lifecycle invoked when exiting the full-screen view
    export function unmount() {
        // Teardown per-view observers and cleanup DOM references
    }

    export default { activate, deactivate, mount, unmount };

---

## 4. Runtime API Surface (`pluginApi`)

When `activate(api)` or `mount(container, api)` is executed, MediaVault injects the `pluginApi` bridge:

    interface PluginApi {
        pluginId: string;
        vaultPath: string;
        language: string;
        items: Item[];
        customTypes: CustomTypeDefinition[];
        nativeApi: NativeElectronBridge;

        setLanguage(code: string): void;
        saveItem(payload: ItemSavePayload): Promise<Item>;
        deleteItem(id: string, isSecure?: boolean): Promise<void>;
        refreshVault(): Promise<void>;
        navigate(filterObj: { type: string; category?: string | null; pluginId?: string }): void;
        openItem(targetItem: Item): void;
        downloadCover(imageUrl: string, title?: string): Promise<string | null>;

        registerAutoFill(handler: AutoFillHandler): void;
        registerSlotView(slotId: string, renderFn: (container: HTMLElement) => void): void;
        registerFloatingAction(actionCfg: UiActionConfig): void;
        registerToolbarButton(actionCfg: UiActionConfig): void;
        registerDetailButton(actionCfg: DetailActionConfig): void;

        registerLanguage(langPayload: CustomLanguagePayload): boolean;
        unregisterLanguage(code: string): void;
        getBaseTranslations(): Record<string, any>;
    }

### Properties
* **`api.pluginId`** (`string`): The active plugin's `id` from `plugin.json`.
* **`api.vaultPath`** (`string`): Absolute filesystem path of the mounted vault root.
* **`api.language`** (`string`): Active client language code (e.g., `'en'`, `'es'`, `'ja'`).
* **`api.items`** (`Array<Item>`): Read-only in-memory snapshot of currently indexed public vault items.
* **`api.customTypes`** (`Array<CustomTypeDefinition>`): User-configured media categories and sections (each supporting any valid Lucide icon name in `icon`).
* **`api.nativeApi`** (`object`): Direct interface to `window.api`.

### Library & Navigation Methods

#### `api.saveItem(payload)`
Creates or overwrites a markdown note with Frontmatter metadata in the vault (automatically sanitized and persisted without `undefined` values):

    await api.saveItem({
        id: `item_${Date.now()}_abc12`,
        filePath: 'C:/Vault/Series/Anime/Frieren.md',
        metadata: {
            title: 'Frieren',
            type: 'Series',
            category: 'Anime',
            status: 'completed',
            rating: 5,
            favorite: true,
            release_year: '2023',
            creator: 'Kanehito Yamada',
            studio: 'Madhouse',
            cover_local: '.assets/frieren_17300000.jpg',
            tags: ['Fantasy', 'Adventure'],
            synopsis: 'After a ten-year quest alongside the hero Himmel...',
            personal_review: 'Masterpiece.',
            seasons: [
                { id: 's_1', number: 1, name: 'Season 1', current: 28, total: 28 }
            ],
            custom_fields: [
                { id: 'field_1', key: 'Audio', value: 'Japanese' }
            ],
            created_at: new Date().toISOString(),
            updated_at: new Date().toISOString(),
            progress: { mode: 'episodes', current: 28, total: 28, unit: 'episode' }
        },
        content: 'Masterpiece.',
        isSecure: false
    });

#### `api.deleteItem(id, isSecure)`
Deletes a vault item by file path or ID:

    await api.deleteItem('C:/Vault/Series/Anime/Frieren.md', false);

#### `api.refreshVault()`
Forces MediaVault to rescan files from disk and refresh its internal cache:

    await api.refreshVault();

#### `api.navigate(filterObj)` & `api.openItem(targetItem)`
Programmatically switches the active workspace view or opens a specific item's detail sheet:

    // Open this plugin's full-screen tool view
    api.navigate({ type: 'plugin-view', pluginId: api.pluginId });

    // Open a specific item in DetailView
    const target = api.items.find((i) => i.metadata?.title === 'Frieren');
    if (target) api.openItem(target);

#### `api.downloadCover(imageUrl, title)`
Downloads a remote image through an HTTP/HTTPS stream directly into `<VaultPath>/.assets/`. Handles redirects, derives extension types (`.jpg`, `.jpeg`, `.png`, `.webp`, `.gif`, `.svg`), formats file names safely, and prevents collisions. Returns the relative path to the image (e.g., `".assets/frieren_1730000000000.jpg"`), or `null` if the request fails.

### UI & Extension Hooks

#### `api.registerSlotView(slotId, renderFn)`
Registers a render function to paint inside dedicated application regions. Full-screen view targeting requires registering the slot `${plugin.id}-view`:

    api.registerSlotView('list-transfer-view', (container) => {
        container.innerHTML = `<div>Custom Interface</div>`;
    });

#### `api.registerAutoFill(handler)`
Registers an asynchronous metadata scraper invoked when creating new catalog records in `NewItemModal`:

    api.registerAutoFill(async (title, selectedType) => {
        const meta = await fetchRemoteMetadata(title, selectedType);
        return {
            title: meta.title,
            release_year: meta.year,
            creator: meta.author,
            studio: meta.studio,
            synopsis: meta.synopsis,
            tags: meta.genres,
            cover_url: meta.poster
        };
    });

#### `api.registerFloatingAction`, `api.registerToolbarButton` & `api.registerDetailButton`
Injects interactive action buttons into the bottom-right floating dock, `TopBar`, or `DetailView` header (using any valid Lucide icon name):

    // Bottom-right floating action pill
    api.registerFloatingAction({
        id: 'quick-import-fab',
        label: 'Quick Import',
        icon: 'Upload',
        onClick: () => api.navigate({ type: 'plugin-view', pluginId: api.pluginId })
    });

    // TopBar action button
    api.registerToolbarButton({
        id: 'sync-toolbar-btn',
        label: 'Sync List',
        icon: 'RefreshCw',
        onClick: () => api.refreshVault()
    });

    // DetailView header button (receives currentItem)
    api.registerDetailButton({
        id: 'export-single-btn',
        label: 'Export Item',
        icon: 'Share',
        onClick: (currentItem) => console.log('Selected:', currentItem.metadata.title)
    });

### Internationalization Extension Hooks

#### `api.registerLanguage(langPayload)` & `api.getBaseTranslations()`
Allows plugins to register new runtime UI languages (built-in ISO codes `en`, `es`, `ja`, `fr`, `de`, `pt` are protected from overrides):

    const baseEn = api.getBaseTranslations().en;
    api.registerLanguage({
        code: 'it',
        name: 'Italiano',
        flag: '🇮🇹',
        translations: {
            ...baseEn,
            sidebar: { ...baseEn.sidebar, dashboard: 'Bacheca' }
        }
    });

---

## 5. Metadata Schema & Disk Layout

MediaVault uses Markdown files with YAML Frontmatter. The normalized object exposed in `api.items` matches the schema below:

### `Item` Data Structure

    {
        "id": "item_1730000000_abc12",
        "filePath": "C:/Vault/Series/Anime/Arcane.md",
        "metadata": {
            "title": "Arcane",
            "type": "Series",
            "category": "Animation",
            "status": "completed",
            "rating": 4.5,
            "favorite": false,
            "release_year": "2021",
            "creator": "Christian Linke",
            "studio": "Fortiche",
            "url": "https://...",
            "cover_url": "https://...",
            "cover_local": ".assets/arcane_1730000000.jpg",
            "tags": ["Sci-Fi", "Steampunk"],
            "synopsis": "Official plot overview...",
            "personal_review": "Personal notes and review...",
            "seasons": [],
            "custom_fields": [],
            "created_at": "2026-01-01T00:00:00.000Z",
            "updated_at": "2026-03-05T12:00:00.000Z",
            "progress": {
                "mode": "episodes",
                "current": 9,
                "total": 9,
                "unit": "episode"
            }
        },
        "content": "Freeform markdown body, notes, or reviews.",
        "isSecure": false
    }

### Filesystem Conventions
Physical items are written using this path structure:

    <VaultPath>/<Type_Folder>/<Category>/<SanitizedTitle>.md

To ensure cross-platform compatibility across Windows and Unix paths, generate target locations using `window.api.joinPath`:

    const safeTitle = `${item.title.replace(/[^a-zA-Z0-9_-]/g, '_')}.md`;
    const physicalPath = await window.api.joinPath(
        api.vaultPath,
        item.type || 'Series',
        item.category || 'General',
        safeTitle
    );

---

## 6. View Routing & UI Mount (`PluginFullView`)

When an active plugin is selected from the Plugin Library ("Open tool" button), the router transitions to `activeFilter.type === 'plugin-view'` and renders the full-width `PluginFullView` container:

    +-------------------------------------------------------------+
    | [<-] Plugin Name  v1.1.0       Description text             |
    +-------------------------------------------------------------+
    |                                                             |
    |   Target Container (<div ref={containerRef} />)             |
    |   - Priority 1: registeredSlotViews.get(`${plugin.id}-view`)|
    |   - Priority 2: instance.module.mount(container, api)       |
    |                                                             |
    +-------------------------------------------------------------+

### Mount Resolution Order
1. **`${plugin.id}-view`**: Checks for a view registered via `api.registerSlotView(`${plugin.id}-view`, fn)`.
2. **`module.mount(container, api)`**: Direct invocation of the exported `mount` function if no slot view was registered for `${plugin.id}-view`.

### Teardown Cleanups
When switching views or unmounting `PluginFullView`:
* MediaVault executes `instance.module.unmount()` if present.
* The container element's inner HTML is cleared (`container.innerHTML = ''`).

---

## 7. Native Electron IPC Bridge (`window.api`)

The bridge exposed under `window.api` (and mirrored at `api.nativeApi`) provides secure, low-level OS operations:

| Method | Signature | Description |
| :--- | :--- | :--- |
| `isStoreBuild` | `boolean` | `true` when running inside a Microsoft Store (`MSIX` / `APPX`) package; `false` on standalone GitHub builds. |
| `scanVault` | `(vaultBasePath: string) => Promise<{ items: Item[], categoryFolders: object }>` | Reads and parses all Frontmatter markdown items and category folders from disk. |
| `joinPath` | `(...paths: string[]) => Promise<string>` | Joins path segments using platform-specific separators (`\` on Windows, `/` on POSIX). |
| `saveItem` | `(payload: { filePath, metadata, content }) => Promise<object>` | Writes Frontmatter and content directly to disk with anti-echo watcher registration. |
| `scanPlugins` | `(vaultBasePath: string) => Promise<DiscoveredPlugin[]>` | Scans `<VaultPath>/.plugins` and returns validated manifests. |
| `loadPluginSource` | `({ vaultBasePath, folderName, file }) => Promise<string \| null>` | Reads text content of the entry JS bundle inside a plugin directory. |
| `loadPluginStyles` | `({ vaultBasePath, folderName }) => Promise<string \| null>` | Reads `styles.css` for a given plugin. |
| `selectPluginZip` | `() => Promise<string \| null>` | Opens a native file dialog to pick a local `.zip` plugin archive. |
| `installPluginZip` | `({ vaultBasePath, zipPath }) => Promise<InstallResult>` | Extracts and installs a local `.zip` plugin into `<VaultPath>/.plugins`. |
| `installRemotePlugin` | `({ vaultBasePath, downloadUrl }) => Promise<InstallResult>` | Downloads and installs a plugin archive from an official registry URL. |
| `uninstallPlugin` | `({ vaultBasePath, folderName }) => Promise<boolean>` | Removes a plugin folder from `<VaultPath>/.plugins`. |
| `downloadPluginAsset` | `({ vaultBasePath, url, title }) => Promise<string \| null>` | Downloads a remote image directly into the vault `.assets/` directory. |
| `openPluginsFolder` | `(vaultBasePath: string) => Promise<boolean>` | Opens the OS native file explorer at `<VaultPath>/.plugins`. |
| `getVaultConfig` | `(vaultBasePath: string) => Promise<VaultConfig>` | Reads configuration data from `.vault-config.json`. |
| `saveVaultConfig` | `({ basePath, config }) => Promise<boolean>` | Persists configuration updates (e.g., `enabledPlugins`) with anti-echo protection. |

---

## 8. Theme System, CSS Variables & Assets

### Dynamic CSS Custom Properties
Custom styling inside `styles.css` should use MediaVault design tokens to support user-selected light, dark, and custom themes:

    /* Card & Surface Tokens */
    var(--bg-main)            /* Primary application background */
    var(--bg-card)            /* Container and card background */
    var(--bg-surface)         /* Input fields, nested panels, badges */
    var(--bg-card-hover)      /* Active hover background */

    /* Accent & Border Tokens */
    var(--accent-primary)     /* Primary accent tint (buttons, active pills) */
    var(--accent-secondary)   /* Secondary accent tint */
    var(--accent-danger)      /* Error badges, delete buttons */
    var(--accent-success)     /* Positive badges, completion states */
    var(--border-subtle)      /* Card outlines, structural borders */
    var(--border-focus)       /* Focused inputs and highlighted drops */

    /* Typography Tokens */
    var(--text-main)          /* Primary text color */
    var(--text-muted)         /* Subtitles, hints, and counters */

### Loading Vault Assets (`vault-asset://`)
To load local images from `<VaultPath>/.assets/`, use the registered protocol:

    <img src="vault-asset:///${api.vaultPath}/.assets/cover_example.jpg" alt="Local Cover" />

---

## 9. Build Configuration & Bundling (`esbuild`)

MediaVault executes `main.js` as an ES2022 module via an in-memory `Blob URL`. Because `Blob URL` imports cannot resolve relative `./` paths or raw `.json` imports at runtime, multi-file plugin codebases (`src/`) **must** be bundled into a single `main.js` file using `esbuild` (or Vite) before packaging into a `.zip` archive.

### Bundler Script (`esbuild.config.mjs`)

    import * as esbuild from 'esbuild';

    const isWatch = process.argv.includes('--watch');

    const buildOptions = {
        entryPoints: ['src/main.js'],
        bundle: true,
        format: 'esm',
        target: 'es2022',
        outfile: 'dist/main.js',
        loader: {
            '.json': 'json'
        }
    };

    if (isWatch) {
        const ctx = await esbuild.context(buildOptions);
        await ctx.watch();
        console.log('Watching plugin source files for changes...');
    } else {
        await esbuild.build(buildOptions);
        console.log('Production build successful: dist/main.js');
    }

### Distributable `.zip` Structure
Package only the compiled output alongside `plugin.json` and `styles.css`:

    my-plugin.zip
    ├── plugin.json
    ├── main.js        (compiled dist/main.js)
    └── styles.css     (optional)

### Manifest & Build Scripts (`package.json`)

    {
        "name": "mediavault-plugin-template",
        "version": "1.0.0",
        "type": "module",
        "scripts": {
            "build": "node esbuild.config.mjs",
            "dev": "node esbuild.config.mjs --watch"
        },
        "devDependencies": {
            "esbuild": "^0.28.2"
        }
    }

---

## 10. Minimal Starter Template

This clean implementation provides an entry point with complete lifecycle handling, custom rendering, and cleanup routines:

    // Entry point for a MediaVault plugin with lifecycle handlers and custom tool view
    let activeInterval = null;

    // Render plugin interface inside target host container
    export function mount(container, api) {
        container.innerHTML = `
            <div style="padding: 1.5rem; color: var(--text-main); font-family: inherit;">
                <h2 style="margin-bottom: 0.5rem;">Plugin Active</h2>
                <p style="color: var(--text-muted); font-size: 0.875rem;">
                    Vault mounted at: <code>${api.vaultPath}</code>
                </p>
                <p style="font-size: 0.875rem;">
                    Indexed public records: <strong>${api.items.length}</strong>
                </p>
                <button id="refresh-btn" style="
                    background: var(--accent-primary);
                    color: var(--bg-main);
                    border: none;
                    border-radius: 0.5rem;
                    padding: 0.5rem 1rem;
                    cursor: pointer;
                    font-weight: 600;
                ">
                    Force Vault Refresh
                </button>
            </div>
        `;

        const button = container.querySelector('#refresh-btn');
        button.addEventListener('click', async () => {
            button.disabled = true;
            await api.refreshVault();
            button.disabled = false;
        });

        return {
            unmount() {
                container.innerHTML = '';
            }
        };
    }

    // Clean up background intervals and DOM listeners
    export function unmount() {
        if (activeInterval) {
            clearInterval(activeInterval);
            activeInterval = null;
        }
    }

    // Initialize plugin slots and optional metadata scraper
    export async function activate(api) {
        api.registerSlotView(`${api.pluginId}-view`, (container) => {
            mount(container, api);
        });

        api.registerAutoFill(async (title, selectedType) => {
            return {
                title: title.trim(),
                category: 'Imported',
                status: 'backlog'
            };
        });
    }

    // Tear down plugin state when disabled
    export async function deactivate() {
        unmount();
    }

    export default {
        activate,
        deactivate,
        mount,
        unmount
    };