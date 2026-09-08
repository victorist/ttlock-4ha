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
4. **Expired Passcode Cleanup Card & Script (`ttlock_delete_card.yaml` and `ttlock_delete_script.yaml`)**: Identifies all passcodes whose validity period has expired and deletes them from the TTLock smart lock, automatically updating the viewer dialog.

---

## 📁 File Structure

* [`ttlock_card.yaml`](ttlock_card.yaml) — PIN creation card configuration powered by `browser_mod.popup`.
* [`ttlock_passcodes_card.yaml`](ttlock_passcodes_card.yaml) — Lock passcode list viewer card configuration powered by `browser_mod.popup`.
* [`ttlock_delete_card.yaml`](ttlock_delete_card.yaml) — Expired passcode deletion card configuration powered by `browser_mod.popup`.
* [`ttlock_script.yaml`](ttlock_script.yaml) — Core Home Assistant script for PIN passcode generation, multi-lock cascading, and verification.
* [`ttlock_list_script.yaml`](ttlock_list_script.yaml) — Home Assistant script for querying and rendering the active passcodes HTML table for a selected lock with a dynamic "Delete Expired" button.
* [`ttlock_delete_script.yaml`](ttlock_delete_script.yaml) — Home Assistant script for identifying, deleting expired passcodes via `ttlock.delete_passcode`, and refreshing the passcode table.
* [`readme.md`](readme.md) — Main English documentation and technical guide.
* [`readme_ru.md`](readme_ru.md) — Russian documentation version.

---

## 🚀 Key Features

* **Passcode List Viewer**: One-click retrieval and display of all active/expired lock passcodes (owner name, PIN code, validity window, status) inside a `browser_mod.popup` dialog.
* **One-Click Expired Passcode Deletion**: Automatically calculates expired passcodes count and presents a `🗑️ Delete Expired (N)` button directly inside the list popup or via a standalone dashboard card.
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

1. Copy the scripts into your Home Assistant configuration (e.g. `scripts.yaml` or UI script editor):
   - [`ttlock_script.yaml`](ttlock_script.yaml) — under key `sozdat_pin_kod_ttlock`
   - [`ttlock_list_script.yaml`](ttlock_list_script.yaml) — under key `ttlock_pokazat_spisok_pin_kodov`
   - [`ttlock_delete_script.yaml`](ttlock_delete_script.yaml) — under key `ttlock_udalit_istekshie_pin_kody`

   *When inserting into `scripts.yaml`, specify the script entity ID key:*
   ```yaml
   ttlock_udalit_istekshie_pin_kody:
     alias: "TTLock — Delete Expired Passcodes"
     ...
   ```
   *When pasting into Home Assistant UI Script Editor (Edit in YAML), **completely clear** the pre-filled template text before pasting.*

2. Add the dashboard card configurations to your Lovelace dashboard:
   - [`ttlock_card.yaml`](ttlock_card.yaml)
   - [`ttlock_passcodes_card.yaml`](ttlock_passcodes_card.yaml)
   - [`ttlock_delete_card.yaml`](ttlock_delete_card.yaml)

3. Reload scripts in Home Assistant via **Developer Tools -> YAML -> Reload Scripts**.

---

### 6. Lock Passcode Viewer & Expired Cleanup (`ttlock_passcodes_card`, `ttlock_list_script` & `ttlock_delete_script`)

For easy inspection and cleanup of passcodes:

1. **Dashboard Cards (`ttlock_passcodes_card.yaml` & `ttlock_delete_card.yaml`)**:
   - Dedicated buttons to open lock selection forms for inspecting passcodes or purging expired ones.

2. **Passcode List & Expired Button (`ttlock_list_script.yaml`)**:
   - Queries stored passcodes via `ttlock.list_passcodes`.
   - Displays passcodes in an HTML table with `🟢 Active` or `🔴 Expired` badges.
   - If expired passcodes are detected (`expired_count > 0`), dynamically adds a red `🗑️ Delete Expired (N)` action button to the modal footer.

3. **Expired Passcode Deletion (`ttlock_delete_script.yaml`)**:
   - Iterates over all detected expired passcode IDs.
   - Calls `ttlock.delete_passcode` for each expired entry.
   - Automatically refreshes and re-renders the updated passcode viewer dialog upon completion.
