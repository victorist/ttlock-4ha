# TTLock Lock Management in Home Assistant

[English Version](readme.md) | [Русская версия](readme_ru.md)

This repository contains a solution for managing, creating, viewing, and deleting temporary and permanent PIN passcodes for TTLock smart locks in Home Assistant. It features automatic device availability verification and explicit passcode validation directly from the lock entries.

---

## 📋 General Overview

TTLock management relies on the `TTLock` integration (the `hass-ttlock` custom component).

The system consists of four main modules:
1. **PIN Creation Dashboard Card (`ttlock_card.yaml`)**: Added to the Home Assistant dashboard. Clicking the button opens a popup PIN creation form powered by **Browser Mod**.
2. **PIN Creation Script (`ttlock_script.yaml`)**: Receives parameters from the form, converts timestamps/dates, checks lock availability, dispatches passcode creation commands, queries passcodes via `ttlock.list_passcodes` for verification, and alerts the user on each lock's status.
3. **Passcode List Viewing Card & Script (`ttlock_passcodes_card.yaml` and `ttlock_list_script.yaml`)**: Allows selecting any lock and displaying a clean HTML table of all registered passcodes, their statuses (`🟢 Active` / `🔴 Expired`), and validity periods.
4. **Expired Passcode Cleanup Card & Script (`ttlock_delete_card.yaml` and `ttlock_delete_script.yaml`)**: Identifies all passcodes whose validity period has expired and deletes them from the TTLock smart lock with real-time live progress tracking and automatic table refresh.

---

## 📁 File Structure

* [`ttlock_card.yaml`](ttlock_card.yaml) — PIN creation card configuration powered by `browser_mod.popup`.
* [`ttlock_passcodes_card.yaml`](ttlock_passcodes_card.yaml) — Lock passcode list viewer card configuration powered by `browser_mod.popup`.
* [`ttlock_delete_card.yaml`](ttlock_delete_card.yaml) — Expired passcode deletion card configuration powered by `browser_mod.popup`.
* [`ttlock_script.yaml`](ttlock_script.yaml) — Core Home Assistant script for PIN passcode generation, multi-lock cascading, and verification.
* [`ttlock_list_script.yaml`](ttlock_list_script.yaml) — Home Assistant script for querying and rendering the active passcodes HTML table for a selected lock with a dynamic "Delete Expired" button.
* [`ttlock_delete_script.yaml`](ttlock_delete_script.yaml) — Home Assistant script for identifying, deleting expired passcodes via `ttlock.delete_passcode` with a live progress counter, and refreshing the passcode table.
* [`readme.md`](readme.md) — Main English documentation and technical guide.
* [`readme_ru.md`](readme_ru.md) — Russian documentation version.

---

## 🚀 Key Features

* **Passcode List Viewer**: One-click retrieval and display of all active/expired lock passcodes (owner name, PIN code, validity window, status) inside a `browser_mod.popup` dialog.
* **One-Click Expired Passcode Deletion with Live Counter**: Automatically calculates expired passcodes count, presents a `🗑️ Delete Expired (N)` button directly inside the list popup, and displays live progress (`Deleted: X of N | Remaining: Y`) during batch deletions.
* **Rate Limiting & Gateway Queue Protection**: Throttled delay between consecutive deletion commands prevents TTLock Cloud API rate limiting and gateway buffer overflows when deleting large passcode batches.
* **Automatic Date/Timestamp Conversion**: Supports ISO date strings as well as millisecond Unix timestamps from `browser_mod`, automatically converting them to `YYYY-MM-DD HH:MM:SS` format.
* **Pre-Execution Availability Checks (Online / Offline)**: Before sending commands, the script verifies lock entity states in Home Assistant (`states(lock_entity) not in ['unavailable', 'unknown']`). If a lock is offline/disconnected from its gateway, passcode creation or deletion is skipped and the user is alerted with a `⚠️ Lock Unavailable` warning.
* **Explicit Passcode Verification (list_passcodes)**: Instead of assuming creation success, the script calls `ttlock.list_passcodes` (returning a `response_variable`), reads active lock passcodes, and confirms the PIN's physical presence.
* **Automatic Cascading for Associated Locks**: Creating a passcode on specific apartment locks automatically duplicates the same PIN code across common entrance and gate locks.

---

## ⚙️ Requirements & Dependencies

1. **TTLock Integration** (`hass-ttlock`): Must be installed and configured in Home Assistant. Provides services:
   * `ttlock.create_passcode` — Dispatches passcode creation via Gateway (`keyboardPwd/add` API).
   * `ttlock.list_passcodes` — Queries actual stored passcodes from lock/cloud (`lock/listKeyboardPwd` API).
   * `ttlock.delete_passcode` — Deletes specified passcode from lock (`keyboardPwd/delete` API).

2. **Browser Mod**: Custom integration enabling popup dialogs (`browser_mod.popup`) and DOM event triggers on dashboard browser windows.

---

## 🛠️ Installation & Setup

### Method 1: Adding Scripts via Home Assistant UI Script Editor (Recommended)

1. Open Home Assistant and navigate to **Settings** -> **Automations & Scenes** -> **Scripts**.
2. Click **Add Script** in the bottom right corner.
3. In the top right corner of the screen, click the **3 dots menu** and select **Edit in YAML**.
4. ⚠️ **IMPORTANT:** Select all existing text (`Ctrl+A` / `Cmd+A`) and **completely clear the pre-filled template** (`sequence: []`, `alias: ...`) so the text area is entirely blank.
5. Copy and paste the complete contents of the script file:
   - [`ttlock_script.yaml`](ttlock_script.yaml) (for PIN creation)
   - [`ttlock_list_script.yaml`](ttlock_list_script.yaml) (for passcode list viewing)
   - [`ttlock_delete_script.yaml`](ttlock_delete_script.yaml) (for expired passcodes deletion)
6. Click **Save**. The script entity IDs will be generated automatically.

---

### Method 2: Adding Scripts via `scripts.yaml` File

If you prefer adding scripts directly into your `scripts.yaml` configuration file:

```yaml
sozdat_pin_kod_ttlock:
  !include ttlock_script.yaml

ttlock_pokazat_spisok_pin_kodov:
  !include ttlock_list_script.yaml

ttlock_udalit_istekshie_pin_kody:
  !include ttlock_delete_script.yaml
```

*Or paste the code directly under a script entity key with a 2-space indentation:*

```yaml
ttlock_udalit_istekshie_pin_kody:
  alias: "TTLock — Delete Expired Passcodes"
  description: "Identifies and deletes all expired passcodes"
  fields:
    ...
  sequence:
    ...
```

After updating `scripts.yaml`, reload them via **Developer Tools** -> **YAML** -> **Reload Scripts**.

---

### Adding Dashboard Cards in Lovelace

1. Open your Home Assistant dashboard.
2. In the top right corner, click the **3 dots menu** -> **Edit Dashboard**.
3. Click **Add Card** (`+`) at the bottom of the screen.
4. Scroll to the very bottom of the card list and select **Manual**.
5. Replace the template YAML with the contents of one of the card files:
   - [`ttlock_card.yaml`](ttlock_card.yaml) — PIN creation button card
   - [`ttlock_passcodes_card.yaml`](ttlock_passcodes_card.yaml) — Passcode list viewer button card
   - [`ttlock_delete_card.yaml`](ttlock_delete_card.yaml) — Expired passcode cleanup button card
6. Click **Save**.

---

### 6. Lock Passcode Viewer & Expired Cleanup Details (`ttlock_passcodes_card`, `ttlock_list_script` & `ttlock_delete_script`)

For easy inspection and cleanup of passcodes:

1. **Dashboard Cards (`ttlock_passcodes_card.yaml` & `ttlock_delete_card.yaml`)**:
   - Dedicated buttons to open lock selection forms for inspecting passcodes or purging expired ones.

2. **Passcode List & Expired Button (`ttlock_list_script.yaml`)**:
   - Queries stored passcodes via `ttlock.list_passcodes`.
   - Displays passcodes in an HTML table with `🟢 Active` or `🔴 Expired` badges.
   - If expired passcodes are detected (`expired_count > 0`), dynamically adds a red `🗑️ Delete Expired (N)` action button to the modal footer.

3. **Expired Passcode Deletion (`ttlock_delete_script.yaml`)**:
   - Iterates over all detected expired passcode IDs.
   - Displays a live progress popup with real-time remaining count updates (`Deleted: X of N | Remaining: Y`).
   - Calls `ttlock.delete_passcode` with a 350ms throttle delay to protect against API rate limits and gateway buffer overflows.
   - Pauses for cloud database sync and automatically re-renders the updated passcode viewer dialog upon completion.
