# Research: Buzzer Enabled by Default (T1000-E & WisMesh Tag)

## Problem Statement

The T1000-E and RAK WisMesh Tag ship with the buzzer **enabled by default**.
Users must perform a triple button press to disable it, and the setting is lost
on factory reset or when flashing new firmware that reformats the filesystem.
This is a common pain point across both devices:

- Issue [#1491](https://github.com/meshcore-dev/MeshCore/issues/1491) —
  "buzzer disabled by default" — has community support and collaborator
  (`@recrof`) endorsement.
- Issue [#1472](https://github.com/meshcore-dev/MeshCore/issues/1472) —
  "Option to disable the buzzer of the T1000e" — 10 comments, users report
  having to rediscover the triple-press trick after every reset. One user
  resorted to redefining the buzzer pin in `platformio.ini` to silence it.
- WisMesh Tag users in #1472 report the back "reboot" button gets pressed
  accidentally in a pocket, rebooting the device and re-enabling the buzzer.
- A user in #1491 reports being unable to disable the buzzer at all on fresh
  T1000-E with firmware 1.12.0, noting that "there's a whole _Comprehensive
  guide to silencing T-1000_ article posted online."

---

## How the Buzzer Default Is Set

### 1. Prefs struct initialization (`MyMesh.cpp:810`)

```cpp
// defaults
memset(&_prefs, 0, sizeof(_prefs));     // ← zeroes everything
_prefs.airtime_factor = 1.0;
strcpy(_prefs.node_name, "NONAME");
_prefs.freq = LORA_FREQ;
// ... other explicit overrides ...
// NOTE: _prefs.buzzer_quiet is NOT explicitly set — it stays 0
```

`memset(&_prefs, 0, ...)` zeroes the entire `NodePrefs` struct. Since
`buzzer_quiet` is **not** explicitly overridden afterward, it defaults to `0`.

In the buzzer code, `0` means **not quiet** (i.e. buzzer **enabled**).

### 2. Prefs file loading (`DataStore.cpp:192-200`)

```cpp
void DataStore::loadPrefs(NodePrefs& prefs, double& node_lat, double& node_lon) {
  if (_fs->exists("/new_prefs")) {
    loadPrefsInt("/new_prefs", prefs, node_lat, node_lon);
  } else if (_fs->exists("/node_prefs")) {
    loadPrefsInt("/node_prefs", prefs, node_lat, node_lon);
    savePrefs(prefs, node_lat, node_lon);
    _fs->remove("/node_prefs");
  }
  // If neither file exists → prefs struct retains its memset-zero defaults
}
```

If no prefs file exists (fresh flash, factory reset), `buzzer_quiet` stays `0`
(enabled).

### 3. Buzzer hardware init (`buzzer.cpp:4-16`)

```cpp
void genericBuzzer::begin() {
    #ifdef PIN_BUZZER_EN
      pinMode(PIN_BUZZER_EN, OUTPUT);
      digitalWrite(PIN_BUZZER_EN, HIGH);   // power on buzzer hardware
    #endif
    quiet(false);                          // force buzzer ENABLED
    pinMode(PIN_BUZZER, OUTPUT);
    digitalWrite(PIN_BUZZER, LOW);
    startup();                             // play startup sound immediately
}
```

`begin()` unconditionally enables the buzzer and plays the startup sound
**before** the persisted preference is applied.

### 4. Preference applied after begin (`ui-new/UITask.cpp:580-581`)

```cpp
#ifdef PIN_BUZZER
  buzzer.begin();                              // enables + plays startup beep
  buzzer.quiet(_node_prefs->buzzer_quiet);     // then applies saved pref
#endif
```

Even if the user had saved `buzzer_quiet = 1`, the startup sound still plays
because `begin()` calls `startup()` before the preference is loaded.

---

## The `buzzer_quiet` Field

| Attribute     | Value |
|---------------|-------|
| **Struct**    | `NodePrefs` (`examples/companion_radio/NodePrefs.h:27`) |
| **Type**      | `uint8_t` |
| **Semantics** | `0` = buzzer enabled, `1` = buzzer quiet/disabled |
| **File**      | `/new_prefs` (binary), byte offset 84 |
| **Default**   | `0` (via `memset`) → **enabled** |

The field name `buzzer_quiet` uses inverted logic: the *natural* zero-default
maps to the *undesired* enabled state. This is the root cause — when prefs are
zeroed (fresh device, factory reset, firmware reflash), the buzzer activates.

---

## When the Default Kicks In

| Scenario | Prefs file exists? | `buzzer_quiet` value | Buzzer state |
|----------|-------------------|---------------------|--------------|
| Fresh flash (new device) | No | `0` (memset) | **Enabled** |
| Factory reset (`CMD_FACTORY_RESET`) | No (filesystem formatted) | `0` (memset) | **Enabled** |
| Firmware update (filesystem preserved) | Yes | Saved value | Preserved |
| Firmware update (filesystem reformatted) | No | `0` (memset) | **Enabled** |
| User toggles off + reboot | Yes | `1` (saved) | Disabled |

### Factory reset code path (`MyMesh.cpp:1748-1758`)

```cpp
} else if (cmd_frame[0] == CMD_FACTORY_RESET && memcmp(&cmd_frame[1], "reset", 5) == 0) {
    bool success = _store->formatFileSystem();  // erases ALL files
    if (success) {
      writeOKFrame();
      delay(1000);
      board.reboot();  // on reboot, no prefs file → defaults apply
    }
```

---

## Buzzer Toggle Mechanism

**Triple button press** triggers `toggleBuzzer()` in both UI implementations:

- `ui-new/UITask.cpp:921-935`
- `ui-orig/UITask.cpp:400-413`

The toggle:
1. Flips the in-memory state (`buzzer.quiet(true/false)`)
2. Writes to `_node_prefs->buzzer_quiet`
3. Calls `the_mesh.savePrefs()` to persist to flash

---

## Hardware Details

### T1000-E

| Pin | GPIO | Function |
|-----|------|----------|
| `PIN_BUZZER` / `BUZZER_PIN` | 25 (P0.25) | PWM output to buzzer element |
| `PIN_BUZZER_EN` / `BUZZER_EN` | 37 (P1.5) | Enable pin — power-gates the buzzer circuit |

Defined in `variants/t1000-e/variant.h:133-136`.

The buzzer enable pin controls power to the buzzer driver circuit. When
`_is_quiet = true`, the enable pin is driven LOW (power off). When false, HIGH
(power on). The T1000-E defines `USER_BTN_PRESSED=HIGH` in its
`platformio.ini:13`.

### RAK WisMesh Tag

| Pin | GPIO | Function |
|-----|------|----------|
| `PIN_BUZZER` | 21 (P0.21) | PWM output to buzzer element |
| `PIN_BUZZER_EN` | **not defined** | No power-gate pin |

Defined in `variants/rak_wismesh_tag/variant.h:121` and
`variants/rak_wismesh_tag/platformio.ini:25`.

**Key hardware differences from T1000-E:**

1. **No `PIN_BUZZER_EN`** — the WisMesh Tag has no separate enable/power-gate
   pin. The `#ifdef PIN_BUZZER_EN` blocks in `buzzer.cpp` are no-ops. The
   `quiet()` method only sets the `_is_quiet` flag; it cannot power off the
   buzzer circuit. The PWM pin is still initialized and LOW.

2. **No `USER_BTN_PRESSED` override** — defaults to `LOW`
   (`ui-orig/UITask.cpp:15-16`). The T1000-E overrides this to `HIGH`. This
   means the button polarity and timing characteristics differ between the two
   devices, which is why users report difficulty with the triple-press on one
   device vs. the other.

3. **`BUZZER_EN` vs `PIN_BUZZER_EN` mismatch** — The WisMesh Tag board's
   `powerOff()` method (`RAKWismeshTagBoard.h:44`) checks `#ifdef BUZZER_EN`,
   but the variant only defines `PIN_BUZZER` (not `BUZZER_EN`). This means the
   buzzer-disable-on-poweroff code is dead code on the WisMesh Tag. (The
   T1000-E defines both `BUZZER_EN` and `PIN_BUZZER_EN` in its `variant.h`.)

4. **Physical reset button** — The WisMesh Tag has a back button (`PIN_BUTTON2`
   = GPIO 12, `variant.h:81-82`) which can be accidentally pressed in a pocket,
   rebooting the device and re-enabling the buzzer if no prefs file exists.

### Shared Code Paths

Both devices use the identical companion radio code:
- Same `NodePrefs` struct, same `DataStore` load/save, same `MyMesh` defaults
- Same `buzzer.begin()` → `buzzer.quiet(prefs)` init sequence in UITask
- Same triple-press toggle in `handleButtonTriplePress()`
- Same factory reset via `CMD_FACTORY_RESET` → `formatFileSystem()`

### PlatformIO Environments with Buzzer Enabled

**T1000-E:**
- `t1000e_companion_radio_usb`
- `t1000e_companion_radio_ble`

**WisMesh Tag:**
- `RAK_WisMesh_Tag_companion_radio_usb` (includes `buzzer.cpp`, line 77)
- `RAK_WisMesh_Tag_companion_radio_ble` (includes `buzzer.cpp`, line 101)

---

## Existing Community Work

### Open Issues

- **[#1491](https://github.com/meshcore-dev/MeshCore/issues/1491)** —
  "[Feature Request] buzzer disabled by default" — directly requests this
  change. Collaborator `@recrof` endorsed it. Author `@446564` offered to
  implement. 6 comments. A user reports being completely unable to disable the
  buzzer on a fresh T1000-E (1.12.0). Another user (`@afourney`) asks how to
  disable it on WisMesh Tag after updating to 1.12.0.

- **[#1020](https://github.com/meshcore-dev/MeshCore/issues/1020)** — "DND mode
  for buzzers" — related request for a do-not-disturb mode, still open.

- **[#1759](https://github.com/meshcore-dev/MeshCore/issues/1759)** —
  "(Emergency) Buzzer/Alert" — feature request for emergency buzzer, tangential.

### Open PRs (related)

- **[#1543](https://github.com/meshcore-dev/MeshCore/pull/1543)** — "More
  Granular Buzzer Controls" — adds bitmask semantics to `buzzer_quiet` (bit 0 =
  master disable, bit 1 = serial-connected disable). Does **not** change the
  default. Would need to be coordinated with if we change the default. Author
  (`@nakoeppen`) is eager to get it merged for 1.14.

- **[#1501](https://github.com/meshcore-dev/MeshCore/pull/1501)** — "Acoustic
  feedback when buzzer is toggled" — UX improvement, orthogonal to default
  change.

### Closed (relevant)

- **[#1130](https://github.com/meshcore-dev/MeshCore/pull/1130)** (merged) —
  "Added buzzer config persistence across restart" — the PR that added
  `buzzer_quiet` persistence. Before this, the buzzer state was lost on every
  reboot.

- **[#1472](https://github.com/meshcore-dev/MeshCore/issues/1472)** (closed) —
  "[Feature Request] Option to disable the buzzer of the T1000e" — 10 comments.
  Users describe the triple-press timing as "critical" and request visual/audio
  feedback. One user (`@rifkegribenes`) specifically notes the WisMesh Tag
  preference doesn't survive accidental reboots from the back button. Another
  user (`@MisterCodeRalf`) proposed code for double-blink LED feedback and
  acknowledgment tones.

- **[#1438](https://github.com/meshcore-dev/MeshCore/issues/1438)** (closed) —
  "RAK Wismesh Tag: Companion firmware doesn't respect notification settings
  when disconnected" — clarifies that notification settings in the companion app
  control phone notifications, not the device buzzer. The device buzzer is only
  controlled by triple-press.

---

## Root Cause Summary

The buzzer is enabled by default due to **two compounding factors**:

1. **`memset(&_prefs, 0, ...)` zeros `buzzer_quiet` to `0`**, and `0` means
   "not quiet" (enabled). There is no explicit default override for
   `buzzer_quiet` like there is for `gps_enabled` or `airtime_factor`.

2. **`buzzer.begin()` unconditionally calls `quiet(false)` and `startup()`**
   before the persisted preference is loaded. Even a device with a saved
   `buzzer_quiet = 1` will play the startup sound.

The fix would need to address both: the default value in the prefs
initialization, and the ordering in `buzzer.begin()` vs preference loading.

---

## Open PR Analysis & Opportunities

### PR #1543 — "More Granular Buzzer Controls" (@nakoeppen)

**What it does:**
- Introduces a bitmask for `buzzer_quiet`: bit 0 = master disable
  (`BUZZER_QUIET_ALWAYS`), bit 1 = disable-when-serial-connected
  (`BUZZER_QUIET_ON_SERIAL`).
- Adds a `BUZZER_QUIET` compile-time default macro (defaults to `1` = quiet) in
  `MyMesh.h`.
- Explicitly sets `_prefs.buzzer_quiet = BUZZER_QUIET` in the `MyMesh`
  constructor alongside the other pref defaults — **this directly fixes the
  default-enabled problem**.
- Adds `constrain(_prefs.buzzer_quiet, 0, 3)` after loading prefs to sanitize
  invalid values.
- Adds `playNotification()` method to both `ui-orig` and `ui-new` that checks
  the bitmask before playing sounds.
- Modifies `toggleBuzzer()` to use XOR bit-flip on bit 0 instead of simple
  boolean toggle.
- Adds `toggleBuzzerOnSerial()` for bit 1 (no UI trigger yet, intended for
  client app control).
- Removes the `!_serial->isConnected()` guard from `queueMessage()` and
  `onDiscoveredContact()`, moving that logic into `playNotification()` instead.

**What it does NOT do:**
- Does not touch `buzzer.cpp` — the `begin()` method still unconditionally calls
  `quiet(false)` and `startup()`. The startup beep still plays even when the
  user has buzzer disabled. The preference is only applied afterward.
- Does not address the `BUZZER_EN` / `PIN_BUZZER_EN` naming mismatch in the
  WisMesh Tag `powerOff()`.
- Has no LED feedback for muted state (that's PR #1501's territory).
- Contains a commented-out alternative `toggleBuzzer()` that cycles through all
  4 states — left for community discussion.

**Status:** Open, no reviews, no merge conflicts reported. Author is eager to
get it merged for 1.14 but has had no maintainer response. Based on `dev` branch
with head branch confusingly also named `dev` (on the fork).

### PR #1501 — "Acoustic feedback when Buzzer is toggled" (@MisterCodeRalf)

**What it does:**
- Adds acoustic feedback on triple-press: plays a rising tone when enabling
  buzzer, a falling tone (with blocking wait) when disabling.
- Adds double-blink LED pattern when buzzer is muted — single blink = buzzer on,
  double blink = buzzer off. Provides persistent visual indicator of mute state.
- Only modifies `ui-orig/UITask.cpp` (the UI used by T1000-E and WisMesh Tag
  companion radio).

**What it does NOT do:**
- Does not change the default value of `buzzer_quiet`.
- Does not touch `ui-new/UITask.cpp`.
- Does not introduce granular bitmask controls.
- Does not fix the startup beep when buzzer is set to quiet.

**Status:** Open, no reviews, no comments. Branch:
`double-blink-buzzer-off2`. Based on `dev`.

### Overlap & Conflicts Between PRs

Both PRs modify `ui-orig/UITask.cpp`'s `notify()` and `handleButtonTriplePress()`
methods. They would conflict textually if both merged, but are conceptually
complementary:
- #1543 refactors the notification gating logic (bitmask-based)
- #1501 adds UX feedback (tones + LED pattern)

Neither PR addresses the `buzzer.begin()` startup sound issue.

### What a New PR Could Do (Without Duplicating Existing Work)

The two open PRs cover the default value (#1543) and toggle UX feedback (#1501).
The remaining gaps are:

#### Gap 1: Startup beep ignores saved preference

`buzzer.begin()` calls `quiet(false)` then `startup()` before UITask loads the
saved preference. Even with PR #1543's default fix, a user who explicitly
disabled the buzzer will still hear the startup beep on every boot.

**Fix:** Refactor `buzzer.begin()` to accept the initial quiet state, or move
`startup()` out of `begin()` and into UITask after the preference is loaded:

```cpp
// Option A: Pass initial state to begin()
void genericBuzzer::begin(bool start_quiet) {
    #ifdef PIN_BUZZER_EN
      pinMode(PIN_BUZZER_EN, OUTPUT);
    #endif
    pinMode(PIN_BUZZER, OUTPUT);
    digitalWrite(PIN_BUZZER, LOW);
    quiet(start_quiet);
    if (!start_quiet) startup();
}

// UITask would then call:
buzzer.begin(_node_prefs->buzzer_quiet);
// No separate buzzer.quiet() call needed
```

```cpp
// Option B: Remove startup() from begin(), let UITask control it
void genericBuzzer::begin() {
    #ifdef PIN_BUZZER_EN
      pinMode(PIN_BUZZER_EN, OUTPUT);
    #endif
    quiet(true);  // start quiet, let caller enable
    pinMode(PIN_BUZZER, OUTPUT);
    digitalWrite(PIN_BUZZER, LOW);
}

// UITask would then call:
buzzer.begin();
buzzer.quiet(_node_prefs->buzzer_quiet);
if (!buzzer.isQuiet()) buzzer.startup();
```

Option A is cleaner — single call, no intermediate state. Option B preserves
the current call structure but adds a conditional.

Both options need changes in both `ui-orig/UITask.cpp` and
`ui-new/UITask.cpp` to match.

#### Gap 2: WisMesh Tag `BUZZER_EN` / `PIN_BUZZER_EN` mismatch

`RAKWismeshTagBoard.h:44` checks `#ifdef BUZZER_EN` but the variant defines
`PIN_BUZZER` (no `BUZZER_EN`). This is dead code — the buzzer is never
disabled during `powerOff()` on the WisMesh Tag. However, since the WisMesh
Tag has no physical enable pin anyway, the practical impact is zero. The dead
code is just misleading. This could be noted in a PR but is low priority.

#### Gap 3: No CLI command for buzzer control

Users without physical button access (serial-only setups, or those who
struggle with triple-press timing) have no way to toggle the buzzer. A `/buz`
CLI command was mentioned by a user in #1472 but does not exist. This is a
separate feature and out of scope for the default-fix PR.

### Recommendation

**Support strategy:**

1. **Comment on PR #1543** acknowledging the default fix and offering to help
   resolve merge conflicts or address the startup beep gap. The PR's
   `BUZZER_QUIET` default macro and explicit `_prefs.buzzer_quiet = BUZZER_QUIET`
   init line directly solve the core issue. Coordinate rather than duplicate.

2. **Submit a focused PR for the startup beep fix** (Gap 1 above). This is
   complementary to #1543 — it fixes a problem #1543 doesn't address, and does
   not conflict with its changes. The PR would touch `buzzer.cpp`, `buzzer.h`,
   and both UITask files — files #1543 mostly doesn't modify (it only adds
   defines to `buzzer.h`).

3. **Leave PR #1501's territory alone** — LED feedback and toggle tones are
   orthogonal UX improvements. Don't duplicate that work.

In summary: the most valuable non-overlapping contribution is fixing
`buzzer.begin()` so the startup sound respects the saved preference. This is a
small, focused change that complements both open PRs.
