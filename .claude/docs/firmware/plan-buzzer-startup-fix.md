# Plan: Fix Startup Beep Ignoring Saved Buzzer Preference

**Related research:** `.claude/docs/research-buzzer-default.md`
**Addresses:** Gap 1 from research — startup beep plays regardless of saved pref
**Complements:** PR #1543 (default value), PR #1501 (toggle UX feedback)

---

## Problem

`buzzer.begin()` unconditionally enables the buzzer and plays the startup sound
before the saved preference is loaded:

```cpp
// buzzer.cpp:4-16 (current)
void genericBuzzer::begin() {
    #ifdef PIN_BUZZER_EN
      pinMode(PIN_BUZZER_EN, OUTPUT);
      digitalWrite(PIN_BUZZER_EN, HIGH);   // powers on buzzer hardware
    #endif
    quiet(false);                          // forces buzzer ENABLED
    pinMode(PIN_BUZZER, OUTPUT);
    digitalWrite(PIN_BUZZER, LOW);
    startup();                             // plays startup sound
}

// ui-orig/UITask.cpp:57-60 (current)
#ifdef PIN_BUZZER
  buzzer.begin();                          // ← beep happens here
  buzzer.quiet(_node_prefs->buzzer_quiet); // ← too late, already beeped
#endif
```

Even when `buzzer_quiet = 1` (saved as disabled), the startup sound plays on
every boot. This affects T1000-E and WisMesh Tag equally.

---

## Approach: Remove `startup()` From `begin()`, Let Caller Control It

Keep `begin()` parameterless (consistent with `MomentaryButton::begin()`,
`GenericVibration::begin()`, and all display `begin()` calls in
`src/helpers/ui/`). Move the startup sound responsibility to the UITask caller
where the preference is already available.

`begin()` becomes purely hardware initialization — configure pins, start quiet.
The caller then applies the saved preference and conditionally plays the startup
sound.

This approach:
- Preserves the existing `begin()` signature — no parameterless `begin()` in
  `helpers/ui/` takes arguments today, and this maintains that convention
- Is compatible with PR #1543 — `buzzer_quiet` is still interpreted solely by
  the caller
- Is compatible with PR #1501 — does not touch `notify()`,
  `handleButtonTriplePress()`, or `userLedHandler()`
- Does not change the default value of `buzzer_quiet` (that's PR #1543's
  territory)

### Compatibility with PR #1543's bitmask

PR #1543 changes `buzzer_quiet` from a simple boolean to a bitmask where bit 0
(`BUZZER_QUIET_ALWAYS`) is the master disable. Our change does not interpret
`buzzer_quiet` at all inside `begin()` — the caller passes it to `quiet()` and
decides whether to call `startup()`. Under #1543, the caller would use
`buzzer_quiet & BUZZER_QUIET_ALWAYS` or just the raw value (non-zero = quiet).
No conflict.

---

## Changes

### 1. `src/helpers/ui/buzzer.cpp` — Initialize quietly, don't play startup

```cpp
// BEFORE (lines 4-16):
void genericBuzzer::begin() {
    #ifdef PIN_BUZZER_EN
      pinMode(PIN_BUZZER_EN, OUTPUT);
      digitalWrite(PIN_BUZZER_EN, HIGH);
    #endif

    quiet(false);
    pinMode(PIN_BUZZER, OUTPUT);
    digitalWrite(PIN_BUZZER, LOW);
    startup();
}

// AFTER:
void genericBuzzer::begin() {
    #ifdef PIN_BUZZER_EN
      pinMode(PIN_BUZZER_EN, OUTPUT);
    #endif
    pinMode(PIN_BUZZER, OUTPUT);
    digitalWrite(PIN_BUZZER, LOW);
    quiet(true);
}
```

**What changed and why:**
- Removed `digitalWrite(PIN_BUZZER_EN, HIGH)` before `quiet()`. The `quiet()`
  method already drives `PIN_BUZZER_EN` HIGH or LOW based on the state. The
  previous code wrote HIGH, then immediately called `quiet(false)` which also
  writes HIGH — redundant. Now `quiet(true)` drives it LOW (disabled).
- `quiet(false)` → `quiet(true)` — hardware starts quiet. The caller will
  enable it after applying the saved preference.
- Removed `startup()` — the caller decides whether to play the startup sound
  based on the user's saved preference.
- GPIO pin mode setup (`pinMode`, `digitalWrite LOW`) is moved before `quiet()`
  to ensure the PWM pin is in a known state before the enable pin is driven.

### 2. `examples/companion_radio/ui-orig/UITask.cpp` — Apply pref, then play

```cpp
// BEFORE (lines 57-60):
#ifdef PIN_BUZZER
  buzzer.begin();
  buzzer.quiet(_node_prefs->buzzer_quiet);
#endif

// AFTER:
#ifdef PIN_BUZZER
  buzzer.begin();
  buzzer.quiet(_node_prefs->buzzer_quiet);
  if (!buzzer.isQuiet()) buzzer.startup();
#endif
```

`begin()` initializes hardware in quiet state. `quiet()` applies the saved
preference. `startup()` only plays if the preference says buzzer is enabled.

### 3. `examples/companion_radio/ui-new/UITask.cpp` — Same change

```cpp
// BEFORE (lines 579-582):
#ifdef PIN_BUZZER
  buzzer.begin();
  buzzer.quiet(_node_prefs->buzzer_quiet);
#endif

// AFTER:
#ifdef PIN_BUZZER
  buzzer.begin();
  buzzer.quiet(_node_prefs->buzzer_quiet);
  if (!buzzer.isQuiet()) buzzer.startup();
#endif
```

---

## Files Modified

| File | Change |
|------|--------|
| `src/helpers/ui/buzzer.cpp` | `begin()` starts quiet, no `startup()` call |
| `examples/companion_radio/ui-orig/UITask.cpp` | Add conditional `startup()` after pref is applied |
| `examples/companion_radio/ui-new/UITask.cpp` | Add conditional `startup()` after pref is applied |

`buzzer.h` is **not** modified — the `begin()` signature stays the same.

---

## Conflict Analysis with Open PRs

| File | PR #1543 | PR #1501 | This PR |
|------|----------|----------|---------|
| `buzzer.h` | Adds `BUZZER_QUIET_*` defines before class | No change | No change |
| `buzzer.cpp` | No change | No change | Modifies `begin()` body |
| `ui-orig/UITask.cpp` | Modifies `notify()`, `toggleBuzzer()`, adds `playNotification()` | Modifies `notify()`, `handleButtonTriplePress()`, `userLedHandler()` | Modifies only `begin()` block (lines 57-60) |
| `ui-new/UITask.cpp` | Modifies `notify()`, `toggleBuzzer()`, adds `playNotification()`, `toggleBuzzerOnSerial()` | No change | Modifies only `begin()` block (lines 579-582) |
| `MyMesh.cpp` | Adds `_prefs.buzzer_quiet = BUZZER_QUIET`, modifies notification logic | No change | No change |
| `MyMesh.h` | Adds `BUZZER_QUIET` macro | No change | No change |
| `NodePrefs.h` | Adds comment to `buzzer_quiet` | No change | No change |

**No textual conflicts.** This PR touches code regions that neither #1543 nor
#1501 modify. The `buzzer.cpp` file is not touched by either open PR, and the
UITask changes are in the `begin()` initialization block (lines 57-60 in
ui-orig, lines 579-582 in ui-new) which neither PR modifies.

---

## Verification

- Build `t1000e_companion_radio_ble` to confirm T1000-E compiles
- Build `RAK_WisMesh_Tag_companion_radio_ble` to confirm WisMesh Tag compiles
- Trace through behavior:
  - Fresh device (no prefs file): `buzzer_quiet = 0` (memset) → `begin()`
    starts quiet → `quiet(0)` enables buzzer → `isQuiet()` returns false →
    `startup()` plays. **Same as current.**
  - User disabled buzzer, reboots: `buzzer_quiet = 1` (loaded from prefs) →
    `begin()` starts quiet → `quiet(1)` keeps buzzer disabled → `isQuiet()`
    returns true → `startup()` skipped. **Fixed — currently plays startup.**
  - Factory reset: prefs erased → `buzzer_quiet = 0` → same as fresh device.
    **Same as current.** (Changing the default to quiet is PR #1543's scope.)

---

## TODO

### Phase 1: Implementation

- [x] 1.1 Create feature branch from `origin/dev`
- [x] 1.2 Modify `src/helpers/ui/buzzer.cpp` — remove `quiet(false)` and
      `startup()` from `begin()`, start quiet, reorder pin init
- [x] 1.3 Modify `examples/companion_radio/ui-orig/UITask.cpp` — add conditional
      `startup()` after `buzzer.quiet()` in the `begin()` block
- [x] 1.4 Modify `examples/companion_radio/ui-new/UITask.cpp` — same change

### Phase 2: Verification

- [x] 2.1 Build `t1000e_companion_radio_ble` — confirm compiles
- [x] 2.2 Build `RAK_WisMesh_Tag_companion_radio_ble` — confirm compiles
- [x] 2.3 Trace through all three scenarios (fresh device, saved-quiet reboot,
      factory reset) and confirm expected behavior in code
- [x] 2.4 Flash `t1000e_companion_radio_ble` firmware.zip to physical T1000-E
      and confirm:
      (a) fresh boot after flash — startup beep plays (default is enabled) ✓
      (b) triple-press to disable buzzer, reboot — startup beep is silent (the fix) ✓
      (c) triple-press to re-enable buzzer, reboot — startup beep plays again ✓

### Phase 3: Submit

- [x] 3.1 Commit changes with GPG signing
- [x] 3.2 Create PR against `meshcore-dev/MeshCore` `dev` branch, referencing
      issue #1491 and noting compatibility with PR #1543
      → PR #1803: https://github.com/meshcore-dev/MeshCore/pull/1803
