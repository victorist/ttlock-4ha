# TTLock Lock Management in Home Assistant

[English Version](file:///Users/vistratov/dev_ha/ttlock/readme.md) | [Русская версия](file:///Users/vistratov/dev_ha/ttlock/readme_ru.md)

This repository contains a solution for managing and generating temporary PIN passcodes for TTLock smart locks in Home Assistant. It features automatic device availability verification and explicit passcode validation directly from the lock entries.

---

## 📋 General Overview

TTLock management relies on the `TTLock` integration (the `hass-ttlock` custom component).

The system consists of two main components:
1. **Control Dashboard Card (`ttlock_card.yaml`)**: Added to the Home Assistant dashboard. Clicking the button opens a popup PIN creation form powered by **Browser Mod**.
2. **PIN Passcode Script (`ttlock_script.yaml`)**: Receives parameters from the form, converts timestamps/dates, checks lock availability, dispatches passcode creation commands, queries passcodes via `ttlock.list_passcodes` for verification, and alerts the user on each lock's status.

---

## 📁 File Structure

* [`ttlock_card.yaml`](file:///Users/vistratov/dev_ha/ttlock/ttlock_card.yaml) — PIN creation card configuration powered by `browser_mod.popup`.
* [`ttlock_passcodes_card.yaml`](file:///Users/vistratov/dev_ha/ttlock/ttlock_passcodes_card.yaml) — Lock passcode list viewer card configuration powered by `browser_mod.popup`.
* [`ttlock_script.yaml`](file:///Users/vistratov/dev_ha/ttlock/ttlock_script.yaml) — Core Home Assistant script for PIN passcode generation, multi-lock cascading, and verification.
* [`readme.md`](file:///Users/vistratov/dev_ha/ttlock/readme.md) — Main English documentation and technical guide.
* [`readme_ru.md`](file:///Users/vistratov/dev_ha/ttlock/readme_ru.md) — Russian documentation version.

---

## 🚀 Key Features

* **Automatic Date/Timestamp Conversion**: Supports ISO date strings as well as millisecond Unix timestamps from `browser_mod`, automatically converting them to `YYYY-MM-DD HH:MM:SS` format.
* **Pre-Execution Availability Checks (Online / Offline)**: Before sending commands, the script verifies lock entity states in Home Assistant (`states(lock_entity) not in ['unavailable', 'unknown']`). If a lock is offline/disconnected from its gateway, passcode creation is skipped and the lock receives the status `⚠️ Lock Unavailable`.
* **Explicit Passcode Verification (list_passcodes)**: Instead of assuming creation success, the script calls `ttlock.list_passcodes` (returning a `response_variable`), reads active lock passcodes, and confirms the PIN's physical presence.
* **Cascaded Processing with Pauses**: When creating a PIN for apartments in groups `B14 1xx` or `B14 2xx`, the passcode is automatically duplicated to linked entrance locks (Entrance 1, Entrance 2, Gate) with a fixed **20-second delay** between requests to prevent gateway overload and race conditions.
* **Dual Notification Flow**:
  - **Dynamic Popups (`browser_mod.popup`)** displayed directly in the user's browser.
  - **System Notifications (`persistent_notification.create`)** saving a persistent summary across all processed locks.

---

## 🛠 Operation Statuses

For every processed lock, one of three explicit statuses is reported:
* `✅ Confirmed` — The lock was online, command succeeded, and passcode existence was verified via `list_passcodes`.
* `❌ Not Created` — The lock was online and the command was sent, but the passcode was not found during verification.
* `⚠️ Lock Unavailable` — The lock was in an `unavailable` or `unknown` state (no gateway connection); no commands were dispatched.

---

## 📖 Detailed Technical Documentation & Operating Principles

### 1. Analysis of Stock `hass-ttlock` Services

An audit of the integration source files (`manifest.json`, `services.yaml`, `api.py`, `models.py`) confirmed built-in support for passcode management without modifying integration source code.

#### `manifest.json`
* Integration Name: `TTLock`
* Type: `cloud_polling`
* Uses `config_flow`, depends on `application_credentials`, `webhook`, `pydantic`.

#### `services.yaml`
The integration exposes the `list_passcodes` action:
```yaml
list_passcodes:
  name: List passcodes
  description: Lists all passcodes for the selected lock, including their names, codes, and validity periods.
```
Declared with `supports_response: SupportsResponse.ONLY` (returns response data into `response_variable`).

#### `list_passcodes` Response Payload
In `api.py`, calling `list_passcodes` sends a `lock/listKeyboardPwd` request to the TTLock API. The returned structure contains a list of `Passcode` objects:
* `id` — Unique passcode ID
* `passcode` — The PIN code string
* `name` — Passcode name/label
* `type` — Passcode type
* `start_date` / `end_date` — Validity window
* `expired` — Expiration status flag

---

### 2. Passcode Verification Logic

Creating and verifying a PIN passcode in TTLock are two independent operations:
1. `ttlock.create_passcode` — Dispatches passcode creation via Gateway (`keyboardPwd/add` API).
2. `ttlock.list_passcodes` — Queries actual stored passcodes from lock/cloud (`lock/listKeyboardPwd` API).

#### Why Verification via `list_passcodes` is Essential
Network glitches or disconnected gateways might cause `create_passcode` to fail silently or be bypassed via `continue_on_error: true`. Thus, issuing a command does not guarantee PIN registration in the lock.

Script Execution Diagram:
```text
               Script Execution
                       │
        ┌──────────────┴──────────────┐
        ▼                             ▼
   Lock Online?                  Lock Offline?
        │                             │
        ├──────────────────┐          ▼
        ▼                  ▼      Display ⚠️
 create_passcode       (skip) "Lock Unavailable"
        │
   Delay 5 sec
        │
  list_passcodes
        │
┌───────┴───────┐
▼               ▼
PIN Found    PIN Not Found
▼               ▼
✅              ❌
Confirmed     Not Created
```

---

### 3. Handling Offline / Unavailable Locks

Prior to any API calls, the script checks the target lock entity state in Home Assistant:
```jinja2
main_is_online: "{{ states(lock_entity) not in ['unavailable', 'unknown'] }}"
```

1. **If the lock is unavailable (`unavailable` / `unknown`)**:
   - The script skips `create_passcode` and `list_passcodes` calls for this lock.
   - Assigns the status `⚠️ Lock Unavailable`.
   - Displays a Browser Mod popup informing the user that the lock is offline (gateway disconnected).
   - Execution continues for any remaining locks in the cascade (e.g. entrance doors or gates).

2. **If the lock is available (`online`)**:
   - Executes `ttlock.create_passcode` (`continue_on_error: true`).
   - Waits 5 seconds.
   - Executes `ttlock.list_passcodes` (`continue_on_error: true`).
   - Searches for a matching `target_pin` or `passcode_name`.
   - Upon a match, assigns status `✅ Confirmed`, otherwise `❌ Not Created`.

---

### 4. Lock Cascading & Timing Intervals

To prevent request collisions and avoid TTLock gateway rate limits, fixed **20-second delays** are enforced between processing individual locks.

#### Operating Scenarios:

1. **Locks `B14 1xx` (Entrance 1)**:
   - Primary apartment lock → create and verify.
   - Pause 20 seconds.
   - Entrance 1 lock (`lock.b14_podezd_1` / `lock.borisenko_14_podezd_1`) → create and verify.
   - Final notification summary.

2. **Locks `B14 2xx` (Entrance 2 & Gate)**:
   - Primary apartment lock → create and verify.
   - Pause 20 seconds.
   - Entrance 2 lock → create and verify.
   - Pause 20 seconds.
   - Gate lock (`lock.borisenko_14_kalitka` / `lock.b14_kalitka`) → create and verify.
   - Final notification summary.

3. **All Other Locks**:
   - Process selected primary lock only → create and verify.
   - Final notification summary.

---

### 5. User Notification Methods

The script generates two notification streams:
1. **Popup Dialogs (`browser_mod.popup`)**:
   - Displayed step-by-step in the active user's browser window.
   - Provides live feedback on execution starts, delays, offline statuses, and final results.
2. **Persistent System Notification (`persistent_notification.create`)**:
   - Saves the aggregated outcome across all locks in the Home Assistant notification panel (`notification_id: ttlock_pin_result`).
