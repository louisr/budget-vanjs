# Budget Dashboard — Test Plan

Manual smoke tests for `docs/index.html`, served at `http://localhost:5500/`.
Each section lists the page, what should be on it, and the flows that originate
there. Steps are intentionally narrow so an issue can be pinpointed.

> **Reset between flows**: open DevTools → Application → IndexedDB →
> `budget-dashboard` → `vault` and delete the `current` record (or run the
> **Reset** flow below) before repeating from Flow 1. If a legacy
> `localStorage["budgetDashboardEncrypted"]` entry exists from before the
> IndexedDB migration, delete that too.

---

## 1. Landing page (no vault) — `#/`

**Expected UI**
- Header: title "Live Budget Dashboard" only (no settings gear, no welcome).
- Card "Get started":
  - Paragraph: "Create a new dashboard or restore an encrypted backup."
  - Buttons: **Create a new dashboard** (primary), **Restore a dashboard** (secondary).
  - Muted hint: "Already have a backup? Use Restore a dashboard."
- Footer: "© Cogitatio LLC 2026" + "About this page" link.
- No lock screen, no modals.

**Flows from here**
- 1a. Click **Create a new dashboard** → opens lock screen in *create* mode (Flow 2).
- 1b. Click **Restore a dashboard** → navigates to `#/restore/` (Flow 6).
- 1c. Click **About this page** → navigates to `#about` (Flow 7).

---

## 2. Create vault (lock screen, create mode)

**Expected UI**
- Lock screen overlay with heading "Create a dashboard".
- Paragraph: "This password encrypts all local data. It is never stored and cannot be recovered."
- Inputs: **Password**, **Confirm password**.
- Buttons: **Create dashboard** (primary), **Cancel** (secondary, returns to landing).

**Flows**
- 2a. Submit with mismatched passwords → inline error "Passwords do not match.", overlay stays.
- 2b. Submit with password shorter than 8 chars → "Password must be at least 8 characters."
- 2c. Submit valid matching password ≥ 8 chars → vault is created, overlay closes, dashboard renders empty (Flow 3).
- 2d. Press **Esc** → cancels create mode (only when no vault exists).
- 2e. Click **Cancel** → closes overlay, back to landing.

---

## 3. Empty dashboard (vault exists, no config) — `#/`

**Expected UI**
- Header: title + settings gear.
- Card "Set up your dashboard":
  - Paragraph about localStorage / GitHub Pages.
  - Button **Open settings** (primary).
- Footer with About link.
- No "Welcome, …" greeting yet (config is null).

**Flows**
- 3a. Click **Open settings** → navigates to `#/settings` (Flow 4).
- 3b. Click the gear icon → also navigates to `#/settings`.
- 3c. Reload the page → lock screen appears (Flow 5).

---

## 4. Settings page — `#/settings`

**Expected UI**
- Card "Settings".
- Top-right buttons: **Export data** (only if `config` is non-null), **Back to dashboard**.
- Theme toggle button: text reads "Dark mode" in light theme, "Light mode" in dark theme.
- Form fields: First name (required), Plaid client ID (required), Plaid secret (required first time, optional thereafter).
- If a secret is already encrypted, a muted note appears under the secret field: "An encrypted secret is saved. Leave blank to keep it; type a new value to replace it."
- Standing note: "The Plaid secret is encrypted at rest with a separate envelope and is only decrypted in memory at sync time…"
- **Save settings** button (primary).
- **Reset Dashboard** button (secondary danger), shown on the same row as **Save settings** and right-aligned.
- **Storage** section below the form: usage estimate from `navigator.storage.estimate()` and a **Request persistent storage** button (replaced with a confirmation line once persistence is granted).

**Flows**
- 4a. Submit empty form → required-field validation prevents submit.
- 4b. Submit valid name + client ID + secret → saves config, navigates back to `#/` (dashboard now shows transactions card; Flow 8).
- 4c. Toggle theme → background switches; persists across reload (after re-unlocking).
- 4d. Click **Back to dashboard** → returns to `#/`.
- 4e. Click **Reset Dashboard** → opens reset modal (Flow 9).
- 4f. (Second visit) Save with secret left blank → keeps existing `plaidSecretEnc`.

---

## 5. Unlock vault (lock screen, unlock mode)

**Expected UI**
- Lock screen with heading "Unlock your dashboard".
- Inputs: **Password** only.
- Buttons: **Unlock** (primary), **Clear settings/data** (secondary danger).

**Flows**
- 5a. Submit correct password → overlay closes, dashboard renders restored state.
- 5b. Submit wrong password → inline error "Incorrect password or corrupted data.", input cleared.
- 5c. Click **Clear settings/data** → opens reset modal (Flow 9).

---

## 6. Restore page — `#/restore/`

**Expected UI**
- Card "Restore a dashboard".
- File input (accepts `.json`).
- Buttons: **Restore** (primary), **Cancel** (secondary).

**Flows**
- 6a. Restore without choosing a file → inline warning "Choose a backup file to restore."
- 6b. Restore with a malformed/wrong-schema file → inline warning "Invalid backup file." File input is cleared.
- 6c. Restore with a valid backup → vault is written to IndexedDB, lock screen opens in *unlock* mode (Flow 5) for the restored vault's password. After successful unlock the URL switches from `#/restore/` to `#/`.
- 6d. Click **Cancel** → returns to `#/`.

---

## 7. About page — `#about`

**Expected UI**
- Card "About this page" with three short paragraphs.
- Footer link changes to "Back to dashboard" (`#/`).

**Flows**
- 7a. Click "Back to dashboard" → returns to `#/`.

---

## 8. Dashboard with config — `#/`

**Expected UI**
- Header shows greeting "Welcome, {firstName}!" + gear.
- Card "Recent transactions":
  - Header right: last sync timestamp (only after at least one sync) + **Sync now** link button.
  - **Sync now** is disabled with a tooltip until a `plaidSecretEnc` is saved.
  - Body: list of transactions, or muted "No transactions yet. Connect Plaid to pull data."

**Flows**
- 8a. Click **Sync now** with secret saved → opens sync modal (Flow 10).
- 8b. Click the gear → navigates to `#/settings` (Flow 4).

---

## 9. Reset modal

**Expected UI**
- Modal "Reset your dashboard".
- Warning: "Warning: this action is irreversible. Existing data will not be recoverable unless you have a backup."
- Buttons: **Confirm reset**, **Download backup**, **Close** (header).

**Flows**
- 9a. Click **Confirm reset** → clears the IndexedDB vault (and any leftover legacy `localStorage` entry), closes modal & lock screen, returns to landing (Flow 1).
- 9b. Click **Download backup** → triggers a `.json` download of the current encrypted payload.
- 9c. Click **Close** or press **Esc** → dismisses modal without changes.
- 9d. Click backdrop → dismisses modal.

---

## 10. Sync modal

**Expected UI**
- Modal "Sync transactions".
- Paragraph explaining the password is needed to decrypt the Plaid secret in memory.
- Input: **Password** (required).
- Buttons: **Sync now** (primary; becomes "Syncing…" + disabled while in flight), **Cancel**, **Close** (header).

**Flows**
- 10a. Submit wrong password → inline warning "Incorrect password — could not decrypt the Plaid secret."
- 10b. Submit correct password → modal closes, transactions appear on dashboard, "Last sync" timestamp updates.
- 10c. Press **Esc** / click backdrop / **Cancel** / **Close** → dismisses modal without syncing.

---

## Cross-cutting checks
- **Esc key precedence**: sync modal → reset modal → cancellable create-vault overlay.
- **CSP / console**: every page should load with no console errors.
- **Theme**: dark mode persists across unlock/reload (it's stored inside the encrypted payload).
- **Persistence**: every state-changing action triggers `persistEncrypted()`; reload + unlock should restore identical state.
- **Plaid secret never serialized in plaintext**: after saving, inspect the IndexedDB `vault.current` payload (decrypt with DevTools if needed) to confirm only `plaidSecretEnc` is present, never `plaidSecret`.
