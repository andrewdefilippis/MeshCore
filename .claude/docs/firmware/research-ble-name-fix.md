# Research: T1000-E BLE Device Name Bug

## Problem Statement

A user reported that the BLE advertised device name depends on the node name length:

| Node name length | BLE name observed          |
|-----------------|---------------------------|
| 1–20 chars       | `MeshCore-<nodename>` (correct) |
| 21–22 chars      | Correct but truncated to show only 20 chars of node name |
| 23+ chars        | `T1000-E-BOOT` (wrong — this is the USB bootloader name) |

The problem occurs with any characters (not just special chars).

---

## Root Cause

**The Nordic SoftDevice S140 v7.3.0 imposes a default maximum GAP device name length of 31 bytes.** Any attempt to set a longer name via `sd_ble_gap_device_name_set()` silently fails (returns `NRF_ERROR_DATA_SIZE`), and the Bluefruit library's `setName()` wrapper does not check the return value.

### Why 31 bytes?

The SoftDevice has a configurable device name length via `BLE_GAP_CFG_DEVICE_NAME`, but the Bluefruit library **never configures it** (the call is commented out in `bluefruit.cpp:365–367`):

```cpp
// bluefruit.cpp:364-367
// Device Name
//  varclr(&blecfg);
//  blecfg.gap_cfg.device_name_cfg =
//  VERIFY_STATUS( sd_ble_cfg_set(BLE_GAP_CFG_DEVICE_NAME, &blecfg, ram_start) );
```

Therefore the SoftDevice uses its default:

```cpp
// ble_gap.h:541
#define BLE_GAP_DEVNAME_DEFAULT_LEN  31  // Default max device name length
```

From the Nordic documentation (`ble_gap.h:1460–1463`):
> If the device name is not configured, the default device name will be `BLE_GAP_DEVNAME_DEFAULT`, the maximum device name length will be `BLE_GAP_DEVNAME_DEFAULT_LEN`, vloc will be set to `BLE_GATTS_VLOC_STACK` and the device name will have no write access.

### Failure chain (on `main` branch)

```
SerialBLEInterface::begin(prefix="MeshCore-", name=<node_name>, pin_code)
│
├─ 1. Bluefruit.begin()
│     └─ sd_ble_gap_device_name_set("T1000-E-BOOT")  ← sets GAP name to USB_PRODUCT (12 chars, succeeds)
│
├─ 2. Bluefruit.setName("MeshCore-<node_name>")
│     └─ sd_ble_gap_device_name_set("MeshCore-<node_name>")
│        ├─ strlen ≤ 31  →  NRF_SUCCESS       →  GAP name = "MeshCore-<node_name>"
│        └─ strlen > 31  →  NRF_ERROR_DATA_SIZE (SILENT FAIL) → GAP name stays "T1000-E-BOOT"
│
├─ 3. Bluefruit.ScanResponse.addName()
│     └─ Bluefruit.getName()  →  reads current GAP name
│        └─ sd_ble_gap_device_name_get() returns whatever the GAP name is
│
└─ Result: scanner sees the GAP name (correct or "T1000-E-BOOT")
```

### Exact thresholds

- `BLE_NAME_PREFIX` = `"MeshCore-"` = **9 bytes**
- Max GAP device name (SoftDevice default) = **31 bytes**
- Max node_name that fits: `31 - 9` = **22 characters**
- Node name of 23+ chars: total = `9 + 23` = **32 bytes** → exceeds 31-byte limit → `sd_ble_gap_device_name_set()` fails

This matches the user's report exactly.

### Scan response truncation (secondary effect)

The `addName()` function in the Bluefruit library (`BLEAdvertising.cpp:178–195`) puts the name in the 31-byte scan response packet. The overhead per AD element is 2 bytes (1 length + 1 type). Since `ScanResponse._count = 0` when `addName()` is called (nothing else is added to ScanResponse), the max name that fits fully is `31 - 2 = 29 bytes`.

For node names of 21–22 chars (total 30–31 bytes):
- `sd_ble_gap_device_name_set()` succeeds (30–31 ≤ 31)
- `addName()` detects overflow: `0 + 30 + 2 = 32 > 31`
- Truncates to 29 bytes and uses `BLE_GAP_AD_TYPE_SHORT_LOCAL_NAME`
- Result: "MeshCore-" (9) + first 20 chars of node_name = 29 bytes → matches user's "truncated to 20 chars"

```cpp
// BLEAdvertising.cpp:178-195
bool BLEAdvertisingData::addName(void)
{
  char name[BLE_GAP_ADV_SET_DATA_SIZE_MAX+1];  // 32-byte buffer

  uint8_t type = BLE_GAP_AD_TYPE_COMPLETE_LOCAL_NAME;
  uint8_t len  = Bluefruit.getName(name, sizeof(name));

  if (_count + len + 2 > BLE_GAP_ADV_SET_DATA_SIZE_MAX)  // doesn't fit?
  {
    type = BLE_GAP_AD_TYPE_SHORT_LOCAL_NAME;
    len  = BLE_GAP_ADV_SET_DATA_SIZE_MAX - (_count+2);   // truncate to fit
  }

  VERIFY( addData(type, name, len) );
  return type == BLE_GAP_AD_TYPE_COMPLETE_LOCAL_NAME;
}
```

---

## Key Constants and Limits

| Constant | Value | Source |
|----------|-------|--------|
| `BLE_GAP_DEVNAME_DEFAULT_LEN` | 31 | `ble_gap.h:541` (SoftDevice default max GAP name) |
| `BLE_GAP_DEVNAME_MAX_LEN` | 248 | `ble_gap.h:542` (absolute max, requires config) |
| `BLE_GAP_ADV_SET_DATA_SIZE_MAX` | 31 | `ble_gap.h:246` (max advertising/scan response packet) |
| `CFG_MAX_DEVNAME_LEN` | 32 | `bluefruit_common.h:45` (Bluefruit library internal) |
| `node_name[32]` | 31 chars + NUL | `NodePrefs.h:13` (max user-settable node name) |
| `BLE_NAME_PREFIX` | "MeshCore-" (9 bytes) | `MyMesh.h:67` |
| USB_PRODUCT (T1000-E) | "T1000-E-BOOT" (12 bytes) | `boards/tracker-t1000-e.json:16` |
| `dev_name` buffer | 48 bytes (32+16) | `SerialBLEInterface.cpp:140` |

---

## Current Branch State (`t1000e-ble-name-fix`)

The branch moves `Bluefruit.setName()` to BEFORE `Bluefruit.begin()`. **This approach is flawed** because:

1. `sd_ble_gap_device_name_set()` requires the SoftDevice to be enabled
2. `Bluefruit.begin()` is what enables the SoftDevice (`sd_ble_enable()`)
3. Calling `setName()` before `begin()` means calling `sd_ble_gap_device_name_set()` before `sd_ble_enable()`, which will fail
4. Then `begin()` sets GAP name to "T1000-E-BOOT" (CFG_DEFAULT_NAME)
5. For non-`@@MAC` names, there is no subsequent `setName()` call — the name will **always** be "T1000-E-BOOT"

The `@@MAC` path still works because it calls `setName()` AFTER `begin()`:
```cpp
if (strcmp(name, "@@MAC") == 0) {
    // ... resolve MAC ...
    Bluefruit.setName(dev_name);  // after begin() — this works
}
```

But for user-set node names (non-@@MAC), the branch makes things worse than `main`.

---

## Affected Code Paths

### 1. `SerialBLEInterface::begin()` (`src/helpers/nrf52/SerialBLEInterface.cpp:126`)
The BLE initialization function that combines prefix + node_name and sets the BLE device name.

### 2. `Bluefruit::setName()` (`bluefruit.cpp:523–527`)
```cpp
void AdafruitBluefruit::setName (char const * str)
{
  ble_gap_conn_sec_mode_t sec_mode = BLE_SECMODE_OPEN;
  sd_ble_gap_device_name_set(&sec_mode, (uint8_t const *) str, strlen(str));
  // ^^^ Return value NOT checked! Silent failure for names > 31 bytes.
}
```

### 3. `Bluefruit::begin()` (`bluefruit.cpp:461–463`)
```cpp
// Default device name — always overwrites any previously set name
ble_gap_conn_sec_mode_t sec_mode = BLE_SECMODE_OPEN;
VERIFY_STATUS(sd_ble_gap_device_name_set(&sec_mode,
    (uint8_t const *) CFG_DEFAULT_NAME, strlen(CFG_DEFAULT_NAME)), false);
```

### 4. `BLEAdvertisingData::addName()` (`BLEAdvertising.cpp:178–195`)
Reads the GAP name and adds it to the scan response. Has its own 29-byte truncation logic.

### 5. Node name validation (`src/helpers/CommonCLI.cpp:475–479`)
Sets node names via CLI. Uses `strncpy` with `sizeof(_prefs->node_name)` = 32. No length validation against BLE limits.

### 6. ESP32 variant (`src/helpers/esp32/SerialBLEInterface.cpp:27`)
Uses `BLEDevice::init(dev_name)` — no SoftDevice constraint. ESP32 BLE stack handles longer names without issue.

---

## Summary of Three Distinct Problems

### Problem 1: SoftDevice max_len (root cause — names 23+ chars)
`sd_ble_gap_device_name_set()` fails silently when name > 31 bytes because `BLE_GAP_CFG_DEVICE_NAME` is not configured and the default max is 31.

### Problem 2: Scan response truncation (cosmetic — names 21–22 chars)
`BLEAdvertisingData::addName()` truncates names longer than 29 bytes to fit the 31-byte scan response packet. This is BLE spec behavior and acceptable, but users may not expect it.

### Problem 3: No BLE-aware name validation
`CommonCLI` allows node names up to 31 chars, but only 22 chars work correctly with the "MeshCore-" prefix over BLE. There is no warning or truncation at the point where the name is set.

---

## Related Files

| File | Role |
|------|------|
| `src/helpers/nrf52/SerialBLEInterface.cpp` | NRF52 BLE initialization |
| `src/helpers/nrf52/SerialBLEInterface.h` | NRF52 BLE interface definition |
| `src/helpers/esp32/SerialBLEInterface.cpp` | ESP32 BLE initialization (no issue) |
| `examples/companion_radio/main.cpp:154` | Calls `serial_interface.begin()` |
| `examples/companion_radio/MyMesh.h:67` | Defines `BLE_NAME_PREFIX` |
| `examples/companion_radio/NodePrefs.h:13` | Defines `node_name[32]` |
| `src/helpers/CommonCLI.cpp:475` | CLI node name setter |
| `boards/tracker-t1000-e.json:16` | USB_PRODUCT = "T1000-E-BOOT" |
| **Bluefruit library (read-only):** | |
| `bluefruit.cpp:365–367` | Commented-out `BLE_GAP_CFG_DEVICE_NAME` config |
| `bluefruit.cpp:461–463` | `begin()` sets name to CFG_DEFAULT_NAME |
| `bluefruit.cpp:523–527` | `setName()` — no return value check |
| `BLEAdvertising.cpp:178–195` | `addName()` — scan response truncation logic |
| `ble_gap.h:541` | `BLE_GAP_DEVNAME_DEFAULT_LEN = 31` |

---

## Potential Fix Approaches

### Approach A: Truncate BLE name to 31 bytes in `SerialBLEInterface::begin()`
- Simplest fix — truncate `dev_name` to 31 chars before calling `Bluefruit.setName()`
- Pro: No library changes needed, works within existing SoftDevice constraints
- Con: Long node names get silently truncated in BLE advertising; the scan response will further truncate to 29 visible chars
- Effective max node name for BLE: 22 chars (31 - 9 prefix)

### Approach B: Truncate BLE name + warn/limit in CommonCLI
- Truncate in `begin()` as in A, plus add a warning or hard limit in the CLI when setting names > 22 chars
- Pro: User gets feedback when setting a name that will be truncated
- Con: Changes validation logic; other platforms (ESP32) don't have this constraint

### Approach C: Configure `BLE_GAP_CFG_DEVICE_NAME` with larger `max_len`
- Set `max_len` to 41 (9 prefix + 32 node_name max) via `sd_ble_cfg_set()` before `sd_ble_enable()`
- This requires modifying the Bluefruit library or injecting the config before `Bluefruit.begin()`
- Pro: Supports full node names up to 32 chars at the GAP level; scan response would still truncate to 29 visible chars
- Con: Requires forking/patching the Bluefruit library; uses more SoftDevice RAM

### Approach D: Truncate BLE name to 29 bytes (scan response optimized)
- Truncate to 29 chars (the scan response maximum) instead of 31
- Pro: The advertised name is always complete (COMPLETE_LOCAL_NAME type), no SHORT_LOCAL_NAME truncation
- Con: 2 fewer chars than Approach A (max node name: 20 chars instead of 22)
- Effective max node name for BLE: 20 chars (29 - 9 prefix)

### Recommended: Approach A with elements of B
- Truncate `dev_name` to 31 chars in `begin()` (fixes the crash/fallback)
- Ensure `setName()` is called AFTER `Bluefruit.begin()` (restore main branch order)
- Optionally truncate to 29 for cleaner scan response advertising
- Consider a CLI warning for names that will be BLE-truncated (optional, lower priority)
