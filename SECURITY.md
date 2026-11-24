# Security & Privacy Policy

## Local-First & Zero-Telemetry Promise
**MediaVault** is designed so your data stays exclusively on your device:
- **No Cloud & No Accounts:** All catalogs, covers, themes, and notes are stored locally on your computer in standard files and folders.
- **Zero Telemetry:** MediaVault collects no analytics, usage tracking, crash telemetry, or personal identifiers.
- **External Link Protection:** Opening web links stored inside your notes or custom fields displays a confirmation prompt showing the full destination URL unless you explicitly choose to trust links in that vault.
- **Minimal, User-Controlled Network Access:** Network requests only occur when you explicitly open the Official Plugin Catalog or check for updates (in GitHub Releases builds, a single silent check runs at startup without sending any personal or library data; in Microsoft Store builds, all updates are handled directly by Windows Update).

## Encrypted Private Vault
- Entries stored in your Private Vault are encrypted locally inside `.secure_vault/manifest.enc` using **AES-256-GCM** authenticated encryption derived from your security PIN via PBKDF2-SHA256 (`100,000` iterations).
- When you lock the Private Vault (`Ctrl + L`), close the app, or remain inactive for 10 minutes, decrypted data is cleared from the interface and the 32-byte master key buffer is zero-filled (`0x00`) in volatile RAM.
- Your one-time **Recovery Key** (`VAULT-XXXX-XXXX-XXXX-XXXX`) is the only way to reset a forgotten PIN offline—save it in a safe location outside your vault folder. See [Troubleshooting & Error Handling](TROUBLESHOOTING.md#2-encrypted-vault-secure_vault-errors) for step-by-step recovery instructions.

## Plugins & Extensions
- Plugins run locally inside your vault's `.plugins/` folder and cannot alter the installed application binaries.
- Only install plugins from trusted sources and review their description and requested permissions before enabling them.

## Reporting a Security Issue
If you discover a security or privacy vulnerability, please **do not post it publicly on GitHub**.

Send the details and steps to reproduce to:
- **Security Contact:** [dasgarstudio@gmail.com](mailto:dasgarstudio@gmail.com)

We will respond within 48–72 hours and coordinate a fix before public disclosure.