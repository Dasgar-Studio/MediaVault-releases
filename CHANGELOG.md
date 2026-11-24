# Changelog

All notable changes to **MediaVault** will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

---

## [1.0.0] - 2026-11-XX

### Added
- **Initial Public Release (Microsoft Store & Windows x64):**
  - Available via the **Microsoft Store** (`MediaVault Atlas`) with automatic Windows Update integration, as well as a standalone **Windows Installer** (with silent background updates) and a **Portable Edition** (`MediaVault Edge`) on GitHub Releases.
- **Local-First Markdown Library:**
  - Organizes your entire collection as clean, human-readable Markdown files inside your own folders—no accounts, no subscriptions, and no cloud lock-in.
  - Instant startup via incremental metadata caching, live two-way sync when editing files in external editors, and automatic file relocation whenever you rename titles, sections, or categories.
  - Built-in duplicate protection that warns you before overwriting an existing vault, section, or entry with the same name.
- **Shelves, Dashboard & Kanban Board:**
  - **Realistic Library Shelves:** Browse your collection on physical-style shelves with sorting controls, customizable column density (5 to 10 columns), half-star ratings, quick `+1` / `-1` progress buttons, and automatic gradient covers when no artwork is set.
  - **Executive Dashboard:** Quick-resume spotlight card, live collection counters, and horizontal carousels for in-progress and recently added titles.
  - **Interactive Kanban Board:** Drag and drop entries across customizable status columns with automatic completion dates.
  - **Detailed Media Sheets:** Track episodes, pages, hours, or simple completion toggles, organize multi-season breakdowns and custom fields with drag-and-drop reordering, and switch statuses right from the header.
- **Browser-Style History & Spotlight Search:**
  - Navigate backward and forward between visited views and entries using the top bar buttons, keyboard shortcuts (`Alt + Left/Right`), or mouse side buttons, with automatic prompts to protect unsaved edits.
  - Press `Ctrl + K` anytime for instant Spotlight search, or `Shift + Click` to select multiple entries for bulk actions.
- **Modernized Recycle Bin:**
  - Visual trash view displaying cover previews, search and section filters, restoration destination indicators, and one-click batch restore or permanent deletion.
- **Zero-Knowledge Encrypted Private Vault:**
  - PIN-protected private space secured with **AES-256-GCM** encryption.
  - Generates a one-time offline **Recovery Key** (`VAULT-XXXX-XXXX-XXXX-XXXX`) during setup (with one-click copy and optional `.txt` export) so you can reset a forgotten PIN without losing your private entries.
  - Includes a live session countdown timer, automatic locking after 10 minutes of inactivity, and an instant `Ctrl + L` panic lock.
- **Visual Customization, Backups & Plugins:**
  - **12 Built-In Themes & Custom Palette Editor:** Choose between 6 Light and 6 Dark presets or customize every interface color live.
  - **Searchable Icon Picker:** Pick from over 150 categorized icons (or any Lucide icon name) using instant keyword search when creating or editing sections.
  - **One-Click `.zip` Backups:** Export a complete, portable backup of your entire library at any time.
  - **Extensible Plugin System:** Install community or official add-ons offline via `.zip` drag-and-drop or browse the on-demand Official Plugin Catalog.
  - **6 Languages:** Full interface support in English, Spanish, French, Portuguese, German, and Japanese.