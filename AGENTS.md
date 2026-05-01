# Budget Dashboard — Agent Guide

A single-page Budget Dashboard built with [VanJS](https://vanjs.org/), hosted on GitHub Pages. The app is fully static: no backend, no build step. All user data lives in browser `localStorage`, encrypted client-side with a user-supplied password.

## Repository Layout
- `index.html` — entire app: markup, inline CSS, and the inline ES module that holds all logic.
- `CNAME` — custom domain for GitHub Pages.
- `AGENTS.md` — this file.

There is no `package.json`, bundler, or test runner. Edit `index.html` directly.

## Local Development
- Serve the directory over HTTP (the inline module imports VanJS via `https://`, and `crypto.subtle` requires a secure context). A simple option:
  ```bash
  python3 -m http.server 5500
  ```
- Open `http://localhost:5500/`. There is no hot reload — refresh the page after edits.
- The CSP `meta` tag allows scripts only from `'self'` and `https://cdn.jsdelivr.net`. If you add another CDN, update the CSP.

## Architecture
- **State**: VanJS `van.state` for `route`, `isUnlocked`, `authMode`, `hasEncrypted`, `config`, `data`, theme, and form fields. The derived session `sessionKey` / `sessionSalt` are module-scoped variables (never serialized).
- **Routing**: hash-based — `#/` (dashboard), `#/restore/` (restore flow), `#about` (about page). `syncRouteFromHash` runs on load and on `hashchange`.
- **UI tree**: `header`, `entryCard`, `mainContent` (route switch), `pageFooter`, plus four overlays toggled via body classes:
  - `modal-open` → settings modal
  - `reset-modal-open` → reset confirmation modal
  - `sync-modal-open` → Plaid sync password prompt
  - `auth-modal-open` → lock screen (create / unlock)
- **Reactivity**: prefer function children (`() => ...`) over `van.derive(...)` returning DOM when a node depends on a state, to keep re-renders local and avoid replacing input nodes (which would reset focus / file selections).

## Encryption
- **Outer vault**: PBKDF2(SHA-256, 600 000 iterations) → AES-GCM 256, derived from the user's password + a per-vault random salt.
- Storage key: `localStorage["budgetDashboardEncrypted"]`.
- Stored payload shape:
  ```json
  { "v": 1, "salt": "<base64>", "iv": "<base64>", "data": "<base64 ciphertext>" }
  ```
- `persistEncrypted()` is a no-op if `sessionKey` is missing — safe to call from anywhere.
- `unlockWithPassword()` decrypts, then calls `setAppState()` and removes the `auth-modal-open` class.
- The password is **never** stored. Losing it means losing the data; surface this in any new flows.
- **Inner secret envelope**: the Plaid secret is stored inside the decrypted config as `plaidSecretEnc = { iv, data }` (base64), encrypted with `sessionKey` via `encryptString` / `decryptString`. The plaintext secret is intentionally never assigned to a `van.state`. It is decrypted only inside `handleSyncSubmit` when the user re-enters their password (the sync modal), used for the Plaid call, then dropped.
- `migrateConfig()` upgrades any legacy `config.plaidSecret` (plaintext field) into a `plaidSecretEnc` envelope on the next unlock, then re-persists.
- If you change PBKDF2 parameters, AES mode, or the inner-envelope shape, bump `v` and add a migration path in `unlockWithPassword` / `migrateConfig`.

## Backup / Restore
- Export: `exportEncryptedData()` writes the raw `localStorage` payload to a `.json` download.
- Restore: `handleRestoreFile()` validates the schema (`v === 1`, `salt`/`iv`/`data` strings) before overwriting `localStorage`. Restore-page errors must use `restoreError`, not `authError`.

## Conventions
- Single file, inline styles. Keep selectors scoped via class names; avoid global tag rules beyond what already exists.
- Dark mode via `body.dark`; new components must theme both modes.
- All overlays close on Escape via the global `keydown` listener at the bottom of the script. New overlays should be added to that handler.
- Plaid is currently a stub (`callPlaidSandbox` returns hardcoded transactions). **Never** wire a real Plaid secret directly into the browser — proxy through a server. Even though the secret is encrypted at rest in `plaidSecretEnc`, a compromised script in the page could still observe it during sync.
- Do not introduce build tooling or `npm` dependencies without a strong reason; the project's value proposition is "open the file and it works."

## Verifying Changes
After editing `index.html`:
1. Reload the local server page.
2. Confirm the browser console is free of CSP, parsing, or VanJS errors.
3. Smoke-test the affected flow: create vault → settings (save name + Plaid creds + secret) → sync (re-enter password) → export → reset → restore → unlock.
