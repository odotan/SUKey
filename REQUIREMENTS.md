# SUKey — Requirements (Phase 1)

> **S**omething **U**  **Key** — Something you know · Something you have · Something you are.
> A USB-C euro cylinder lock. Replaces a standard mechanical euro cylinder; opens when a USB stick containing an authorized file is inserted.

This document specifies **Phase 1** only: the core lock with standard, off-the-shelf USB sticks as keys. Phase 2 (SUK hardware key, SIM-card challenge-response, phone app, WiFi/BT, OTA) is listed at the end as out of scope for now.

---

## 1. Concept

The user inserts a USB-C stick into the lock. The lock reads files from the stick, hashes them, and checks the hashes against an internal allow-list. If a hash matches and the associated access rules permit, the lock energizes a solenoid that withdraws a magnetic locking pin. The user can then turn the cylinder. Turning the cylinder mechanically breaks the power circuit, releasing the solenoid; the pin stays out of the way until the cylinder returns to its rest position, at which point the pin drops back into the locked position.

There is **no battery monitoring loop, no idle MCU, no always-on radio**. The board only has power while a USB stick is plugged in (or the inside knob is pressed). This is the security and battery-life design center.

---

## 2. Hardware

| Component | Choice / Notes |
|---|---|
| MCU | Seeed XIAO ESP32-S3 (21 × 17.5 mm). Native USB host. |
| Outside port | USB-C receptacle. No display. |
| Outside indicator | Single tri-color LED (red / yellow / green). |
| Inside port | USB-C receptacle. |
| Inside display | 0.91" monochrome OLED (128×32, I²C, SSD1306-class). |
| Inside input | Knob with push-button (momentary switch). Rotary input optional. |
| Actuator | Solenoid driving a magnetic pin. Driven via GPIO + MOSFET. |
| Storage | ESP32-S3 internal flash only (NVS / LittleFS). No SD card. |
| Power | Internal rechargeable battery (LiPo). |
| Recovery | Hidden tactile button reachable only when the lock body is removed from the door. |

**Power gating (mechanical):**
- Inserting a USB stick into either port **completes** the power circuit and boots the MCU.
- Removing the stick **breaks** the circuit. MCU loses power.
- Rotating the cylinder also breaks the same circuit (intentional — see §6.5).

---

## 3. Roles & Key Types

A "key" is **a file on a USB stick**. The lock identifies a key by the SHA-256 of the file's bytes. The file can be anything — an image a friend sent, a generated random blob, a JSON, etc.

Two key roles exist:

- **User key** — grants access to unlock. Subject to access rules (expiration, time windows, use-count).
- **Master key** — additionally authorizes key management: adding new keys (via the dual-port provisioning flow, §6.4), removing keys, syncing, and viewing logs. A master key is just a user key with `master: true` in its database entry; nothing on the stick distinguishes it.

A USB stick may carry multiple files. Each file is hashed independently; the highest-privilege match wins.

**Port permissions are asymmetric.** The outside port is **read-only with respect to the key database and lock configuration** — it can authenticate, energize the solenoid, decrement use-counters, and append logs, but it cannot add, remove, or edit key entries, nor change settings. All database and configuration mutations require the inside port. Provisioning a new key (§6.4) uses **both** ports simultaneously: master on the outside for authorization, target stick on the inside for the actual write. This keeps "write to board" capability physically restricted to the inside of the door.

---

## 4. File Discovery on the USB Stick

When a stick is inserted, the firmware:

1. Looks for a top-level folder named `key` (case-insensitive: `key`, `Key`, `KEY` all match).
2. **If the folder exists**, hashes every regular file directly inside it. No size limit.
3. **If the folder does not exist**, scans the root of the stick and hashes every regular file ≤ 100 KB. It also descends **one level** into each immediate subdirectory of the root and hashes every regular file in them ≤ 100 KB. It does **not** recurse beyond that.
4. Each file becomes one candidate hash.

Discovery + hashing must complete within:
- **≤ 100 ms per file**
- **≤ 1 s total wall-clock** for the whole authentication step.

If discovery exceeds the total budget, the lock denies and shows red. (We don't want a slow stick to be a side-channel for holding the door open.)

---

## 5. Key Database

Stored in internal flash. JSON document. Conceptual shape (concrete schema in `schemas/`):

- `keys[]` — each entry has:
  - `hash` — SHA-256 hex of the file (lowercase, 64 chars)
  - `name` — human label (shown in inside UI and logs)
  - `master` — bool; if true, this key has master privileges
  - `expiration` — ISO date or `null` for no expiration
  - `timeWindows` — array of `"HH:MM-HH:MM"` strings; empty/absent means any time
  - `useLimitRemaining` — integer or `null` for unlimited; decremented on each grant
  - `createdBy` — `"master"` or hash of the master key that created it
  - `createdDate` — ISO timestamp

The database is the single source of truth for "is this key allowed."

---

## 6. Behaviors

### 6.1 Outside port — authentication

1. USB stick inserted → board powers on.
2. Boot ESP32-S3 (~1–2 s). LED off during boot, then **yellow** as soon as firmware starts processing.
3. Discover and hash files (§4).
4. For each candidate hash, look it up in the key database.
5. If any match is found, evaluate access rules (expiration, time windows, use-count).
6. **Granted** → LED **green**, energize solenoid, decrement `useLimitRemaining` if set, log the event.
7. **Denied** → LED **red**, log the event, do not energize.
8. The user pulls the stick / turns the cylinder; either action cuts power.

The outside port has **no display** and **no buttons**, and the firmware **never mutates the key database or lock configuration from the outside port path** — even for master keys. The only writes performed on an outside-port grant are decrementing `useLimitRemaining` on the matched entry and appending a log entry. Provisioning a new key requires the inside port (§6.4).

### 6.2 Inside port — authentication (default)

Identical to §6.1 with two differences:
- The OLED shows status text mirroring the LED state ("Welcome", "Granted: <name>", "Denied: <reason>").
- If the matched key has `master: true`, the firmware does **not** auto-energize the solenoid. Instead, the master menu (§6.3) is shown immediately with "Open door" as the default highlighted item. The user must press the knob to confirm — either to unlock or to pick a management action. This prevents accidental unlocks when the user only meant to manage keys.

### 6.3 Inside port — master menu

Available only when a master key is the matched key on the inside port. Items:

- **Open door** — same as a normal grant; default selection.
- **Remove keys** — list current entries by `name`; user scrolls/selects via knob; confirm to delete.
- **Sync** — reconcile the lock's key database with the contents of the inserted stick (intended for app-prepared sticks; details in Phase 2). For Phase 1, this is a stub.
- **View logs** — paginated, scroll via knob.

Selection mechanism: knob press to confirm, knob rotation (if available) or repeated press-to-cycle to navigate. **No on-screen text input** — all configuration is either menu-driven or comes from files on the stick.

**Adding** new keys is *not* a master-menu item. It is a separate flow (§6.4) that requires the master on the outside port and the target stick on the inside port — because the master menu is only entered when a master is on the inside port alone.

### 6.4 Provisioning a new key (offline, no app)

Adding a new key requires **two USB sticks inserted simultaneously**: the master key on the **outside** port (proves authorization), and the **target stick** on the **inside** port (the stick that will become the new key). The outside port is read-only; both writes — to the lock's key database and to the target stick — happen via the inside port.

The firmware decides what mode it is in by inspecting **both ports** at the point of authentication, after USB enumeration completes:

- **Only outside active** → normal grant flow (§6.1).
- **Only inside active** → normal grant flow with possible master menu (§6.2 / §6.3).
- **Both active**, outside-port stick matches a master entry → **provisioning mode** (steps below). The solenoid does **not** energize in this mode.
- **Both active**, outside-port stick is not a master → fall back to inside-port flow; provisioning is not offered.

Provisioning steps:

1. The inside display shows a provisioning prompt with editable access-rule fields: expiration (none / 1h / 1d / 1w / custom-date) and a **master-flag checkbox** (default unchecked). The user navigates via the knob.
2. User confirms via knob press.
3. Lock writes a new file (random bytes, e.g., 256 bytes, into a `key/` folder on the inside stick) and adds the SHA-256 of that file to its database with the chosen access rules.
4. Display: *"Key added. Remove sticks."*

If the user pulls either stick before confirming, the flow aborts; nothing is written to the stick or the database. More fields can be added to the prompt later without breaking the schema.

### 6.5 Solenoid timing

- On grant, energize the solenoid GPIO **high**.
- Hold high for **up to 5 s** (configurable upper bound, fixed in firmware for now).
- The user turning the cylinder mechanically breaks the power rail → solenoid de-energizes → MCU dies. Both are fine: the pin is already out of the bore and the cylinder has rotated past its rest position. The pin only re-engages when the cylinder returns to rest.
- If the user does nothing within the 5 s, the firmware de-energizes the solenoid itself (so a USB stick left dangling doesn't drain the battery via the solenoid coil).

### 6.6 Logging

Every event that powers up the MCU produces at least one log entry — including failed authentication attempts, configuration changes, factory resets, and "stick had no candidate files" cases. Stored in internal flash. Conceptual shape (schema in `schemas/`):

- `timestamp`
- `port` — `"outside"` | `"inside"` | `"both"` (provisioning) | `"recovery"` (factory reset)
- `keyHash` — the candidate hash (or `null` if no candidates were found)
- `keyName` — name from DB if matched
- `result` — `"granted"` | `"denied"` | `"config_change"`
- `reason` — `"authorized"` | `"unknown_hash"` | `"expired"` | `"outside_time_window"` | `"use_limit_exceeded"` | `"hash_budget_exceeded"` | `"no_files_found"` | `"key_added"` | `"key_removed"` | `"factory_reset"` | `"first_enrollment"`
- `solenoidActivated` — bool

Logs are stored in a ring buffer sized to fit comfortably in available flash (target: last 1000 entries). Oldest entries are overwritten.

Logs export automatically on **every** master-key insertion (inside port, or as part of the dual-port provisioning flow): the firmware writes a `logs.json` file to the root of the master's stick containing the current ring-buffer contents.

### 6.7 Recovery (lost master key)

A hidden tactile button is accessible only after physically removing the cylinder from the door. Holding this button for ≥ 5 s while powering up performs a **factory reset**: wipes the key database, wipes logs, returns to "no keys enrolled" state.

In the empty state, inserting any stick on the **inside** port shows a first-enrollment prompt with editable fields, including a **"Make this a master key"** checkbox (default unchecked) and the same access-rule fields as §6.4. The user confirms via knob; the lock writes a new key file to the stick and adds its hash to the database with whatever was set. Subsequent enrollments use the normal dual-port provisioning flow (§6.4), which requires an existing master — so a user who skips the master checkbox on the very first enrollment will need another factory reset to add more keys.

This mirrors the security model of a regular cylinder lock: physical access to the cylinder body = ability to re-key.

---

## 7. UI Specification

### 7.1 LED states (outside + inside)

| State | LED |
|---|---|
| Boot / processing | Yellow (steady) |
| Granted | Green (steady, until power cut) |
| Denied | Red (steady, ~1 s, then power cut) |
| Internal error | Red (blinking, ~1 s, then power cut) |

### 7.2 Display states (inside only)

The OLED is small (128×32). Text-only, large font. One line of status, optional second line of detail.

| State | Display |
|---|---|
| Boot | `SUKey` |
| Reading | `Reading...` |
| Granted | `Welcome` / `<name>` |
| Denied | `Denied` / `<reason>` |
| Master menu | `> Open door` / `Add Remove Sync Logs` (selected item highlighted) |
| Add keys, awaiting blank | `Insert blank` / `on outside` |
| Add keys, writing | `Writing...` |
| Add keys, done | `Key added` / `<name>` |

---

## 8. Performance & Resource Budgets

| Target | Value |
|---|---|
| MCU boot to first LED state | ≤ 2 s |
| File hash | ≤ 100 ms per file |
| Total authentication (insert → LED green/red) | ≤ 3 s |
| Key DB capacity | ≥ 200 keys (hard limit by available flash) |
| Log ring capacity | ≥ 1000 entries |
| Solenoid energize duration | ≤ 5 s, auto-cutoff |

---

## 9. Out of Scope (Phase 2+)

The following are explicitly deferred. They are recorded here so we don't forget them, not as Phase 1 requirements:

- **SUK hardware key** — USB-C lanyard cable form factor with internal flash, rechargeable battery (charges the lock on each insert), and two nano-SIM slots wired through to the lock.
- **SIM-card challenge-response** — using ISO 7816 / PCSC (e.g., ACR39U-NF style smart card reader on the lock side) to verify physical possession of two SIM cards as an additional master-key requirement.
- **Biometrics** — fingerprint or voice as a third factor.
- **WiFi / Bluetooth** — optional radio module for remote logs / live status.
- **Phone app** — for advanced configuration, master-key requirement editing, log viewing, key management; communicates with the lock by writing instruction files to a USB stick.
- **OTA / firmware updates** — via instruction file + physical program button (security: requires removing the lock body to enter programming mode).
- **Configurable master-key requirements** — `fileRequired` / `simCardsRequired` / `biometricRequired` toggles editable from the app.
- **AFTP platform** — separate project; documented separately when we get to it.
