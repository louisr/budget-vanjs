# Budget Dashboard — Agent Guide

A single-page Budget Dashboard built with [VanJS](https://vanjs.org/), hosted on GitHub Pages. The app is fully static: no backend, no build step. All user data lives in the browser (IndexedDB, with a one-time migration from legacy `localStorage`), encrypted client-side with a user-supplied password.

## Repository Layout
- `docs/index.html` — entire app: markup, inline CSS, and the inline ES module that holds all logic. This is the document root served by Live Server and (when configured) GitHub Pages.
- `docs/js/van-1.6.0.min.js` — vendored VanJS runtime. Imported by `index.html` as `./js/van-1.6.0.min.js` so we never load JS from a CDN at runtime.
- `docs/CNAME` — custom domain for GitHub Pages.
- `.vscode/settings.json` — pins Live Server's root to `/docs`.
- `AGENTS.md` — this file.

There is no `package.json`, bundler, or test runner. Edit `docs/index.html` directly.

## Local Development
- The recommended workflow is the VS Code Live Server extension; it is configured (via `.vscode/settings.json`) to serve from `docs/`.
- Alternatively, serve the `docs/` directory over HTTP yourself (the inline module imports VanJS as a relative `./js/van-1.6.0.min.js`, and `crypto.subtle` requires a secure context):
  ```bash
  python3 -m http.server 5500 --directory docs
  ```
- Open `http://localhost:5500/`. There is no hot reload — refresh the page after edits.
- The CSP `meta` tag allows scripts and connections only from `'self'`. VanJS is vendored at `docs/js/van-1.6.0.min.js`; do not re-introduce CDN imports without auditing the supply-chain implications and updating the CSP.

## Architecture
- **State**: VanJS `van.state` for `route`, `isUnlocked`, `authMode`, `hasEncrypted`, `config`, `data`, theme, and form fields. The derived session `sessionKey` / `sessionSalt` are module-scoped variables (never serialized).
- **Routing**: hash-based — `#/` (dashboard), `#/settings` (settings), `#/restore/` (restore flow), `#about` (about page). `syncRouteFromHash` runs on load and on `hashchange` and is also responsible for repopulating settings form fields when entering `#/settings`.
- **UI tree**: `header`, `entryCard`, `mainContent` (route switch), `pageFooter`, plus three overlays toggled via body classes:
  - `reset-modal-open` → reset confirmation modal
  - `sync-modal-open` → Plaid sync password prompt
  - `auth-modal-open` → lock screen (create / unlock)
- Modals are reserved for confirmations (reset) and re-auth prompts (sync, unlock). Editing flows live on dedicated routes (e.g. `#/settings`).
- **Reactivity**: prefer function children (`() => ...`) over `van.derive(...)` returning DOM when a node depends on a state, to keep re-renders local and avoid replacing input nodes (which would reset focus / file selections). When a function child branches on multiple states, read each `state.val` unconditionally at the top so VanJS tracks every dependency, not just the ones taken on the first render.

## Encryption
- **Outer vault**: PBKDF2(SHA-256, 600 000 iterations) → AES-GCM 256, derived from the user's password + a per-vault random salt.
- **Storage backend**: IndexedDB database `budget-dashboard`, object store `vault`, record key `current` of shape `{ name: "current", payload: <envelope> }`. A legacy `localStorage["budgetDashboardEncrypted"]` copy is read as a fallback and migrated into IndexedDB on the next successful unlock (and then deleted). The envelope shape is unchanged from v1, so backup `.json` files remain interchangeable.
- Stored envelope shape:
  ```json
  { "v": 1, "salt": "<base64>", "iv": "<base64>", "data": "<base64 ciphertext>" }
  ```
- IndexedDB helpers: `openVaultDb`, `idbGet`, `idbPut`, `idbDelete`, plus high-level `loadStoredVault`, `writeStoredVault`, `clearStoredVault`, `hasStoredVault`. Always use the high-level wrappers from app code so the legacy-localStorage fallback stays consistent.
- `persistEncrypted()` is a no-op if `sessionKey` is missing — safe to call from anywhere. It writes via `writeStoredVault`, which also deletes any leftover legacy `localStorage` entry.
- `unlockWithPassword()` calls `loadStoredVault()` (IDB → legacy fallback), decrypts, calls `setAppState()`, and removes the `auth-modal-open` class. If the vault came from the legacy localStorage path, it re-persists immediately so subsequent loads use IndexedDB.
- The password is **never** stored. Losing it means losing the data; surface this in any new flows.
- **Inner secret envelope**: the Plaid secret is stored inside the decrypted config as `plaidSecretEnc = { iv, data }` (base64), encrypted with `sessionKey` via `encryptString` / `decryptString`. The plaintext secret is intentionally never assigned to a `van.state`. It is decrypted only inside `handleSyncSubmit` when the user re-enters their password (the sync modal), used for the Plaid call, then dropped.
- `migrateConfig()` upgrades any legacy `config.plaidSecret` (plaintext field) into a `plaidSecretEnc` envelope on the next unlock, then re-persists.
- If you change PBKDF2 parameters, AES mode, or the envelope shape, bump `v` and add a migration path in `unlockWithPassword` / `migrateConfig`. The IndexedDB schema version (currently `1`) is separate from the envelope `v` and is bumped via `indexedDB.open(idbName, <version>)`'s `onupgradeneeded`.

## Storage capacity
- IndexedDB quota is browser- and disk-dependent (commonly hundreds of MB to several GB), so the previous ~5 MB `localStorage` ceiling no longer applies.
- The Settings page surfaces `navigator.storage.estimate()` and offers a "Request persistent storage" button (`navigator.storage.persist()`) so users can opt into eviction protection.
- The vault is currently written as a single record. If individual encrypt/decrypt operations become too large, chunk transactions (e.g. one record per year keyed `tx:<year>`) before introducing OPFS.

## Backup / Restore
- Export: `exportEncryptedData()` reads the envelope via `loadStoredVault()` and writes it to a `.json` download.
- Restore: `handleRestoreFile()` validates the schema (`v === 1`, `salt`/`iv`/`data` strings) before writing it via `writeStoredVault()` (which also clears the legacy localStorage key). Restore-page errors must use `restoreError`, not `authError`.

## Conventions
- Single file, inline styles. Keep selectors scoped via class names; avoid global tag rules beyond what already exists.
- Dark mode via `body.dark`; new components must theme both modes.
- All overlays close on Escape via the global `keydown` listener at the bottom of the script. New overlays should be added to that handler.
- Plaid is currently a stub (`callPlaid` returns hardcoded transactions, ignoring `plaidApiBaseUrls[env]`). The Settings page exposes a `plaidEnv` selector (`sandbox` | `development` | `production`) which is persisted on `config` and passed to `callPlaid` at sync time. **Never** wire a real Plaid secret directly into the browser — proxy through a server. Even though the secret is encrypted at rest in `plaidSecretEnc`, a compromised script in the page could still observe it during sync.
- Do not introduce build tooling or `npm` dependencies without a strong reason; the project's value proposition is "open the file and it works."

## Verifying Changes
After editing `docs/index.html`:
1. Reload the local server page.
2. Confirm the browser console is free of CSP, parsing, or VanJS errors.
3. Smoke-test the affected flow: create vault → settings (save name + Plaid creds + secret) → sync (re-enter password) → export → reset → restore → unlock.
