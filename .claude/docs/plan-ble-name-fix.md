# Plan: T1000-E BLE Device Name Fix

## Problem

Node names >= 23 characters cause the BLE device name to show as "T1000-E-BOOT" instead of "MeshCore-\<nodename\>". See [research-ble-name-fix.md](research-ble-name-fix.md) for root cause analysis.

## Goal

Truncate the BLE device name to fit the SoftDevice's 31-byte GAP name limit and the 29-byte scan response limit, while preserving the **beginning and end** of the node name (where users place emoji and device-type identifiers). Use middle-truncation with a `..` separator.

## Design

### Target length

The BLE name is composed of `prefix` + `node_name_portion`. Two hard limits apply:

| Limit | Value | Source |
|-------|-------|--------|
| GAP device name | 31 bytes | `BLE_GAP_DEVNAME_DEFAULT_LEN` (SoftDevice default) |
| Scan response AD | 29 bytes | 31 (`BLE_GAP_ADV_SET_DATA_SIZE_MAX`) - 2 (length + type fields) |

We target **29 bytes total** so the scan response can advertise the name as `COMPLETE_LOCAL_NAME` without the Bluefruit library silently chopping the end off.

With `"MeshCore-"` (9 bytes), that leaves **20 bytes** for the node name portion.

### Middle-truncation algorithm

When `strlen(prefix) + strlen(name) > 29`:

1. Compute `name_budget = 29 - strlen(prefix)` (=20 for "MeshCore-")
2. Separator: `".."` (2 ASCII bytes) — visually clear, minimal cost
3. Content budget: `name_budget - 2` = 18 bytes for head + tail
4. Split evenly: `head_target = content_budget / 2`, `tail_target = content_budget - head_target` (9 / 9 for default prefix)
5. Walk forward from the start of `name`, counting complete UTF-8 characters, until adding the next character would exceed `head_target` bytes → this is `head_len`
6. Walk backward from the end of `name`, counting complete UTF-8 characters, until adding the next character would exceed `tail_target` bytes → this is `tail_len` and `tail_start`
7. Assemble: `prefix` + `name[0..head_len]` + `".."` + `name[tail_start..end]`

### UTF-8 safety

A UTF-8 character's byte length is determined by its leading byte:

| Leading byte pattern | Character length |
|---------------------|-----------------|
| `0xxxxxxx`          | 1 byte (ASCII)  |
| `110xxxxx`          | 2 bytes         |
| `1110xxxx`          | 3 bytes         |
| `11110xxx`          | 4 bytes (emoji) |
| `10xxxxxx`          | continuation — not a leading byte |

The algorithm only advances by complete characters, so it will never split a multi-byte sequence. The existing codebase already uses this pattern (`DisplayDriver.h:52–55`).

### Examples

**Prefix: `"MeshCore-"` (9 bytes), budget for name: 20 bytes, separator: `".."` (2 bytes), content: 18 bytes (9+9)**

| Node name | Bytes | Result | Notes |
|-----------|-------|--------|-------|
| `📱 Andrew KI7PXC T1000-E` | 26 | `MeshCore-📱 Andr..C T1000-E` | Emoji + device type preserved |
| `SuperLongNodeName🤖` | 21 | `MeshCore-SuperLong..eName🤖` | Emoji at end preserved |
| `ABCDEFGHIJKLMNOPQRSTUVW` | 23 | `MeshCore-ABCDEFGHI..OPQRSTUVW` | Pure ASCII, even split |
| `📱📡🧱🤖💀🔥` | 24 | `MeshCore-📱📡..💀🔥` | All emoji, 8+2+8=18 |
| `Vya📡` | 7 | `MeshCore-Vya📡` | No truncation needed |
| `HowlBot🤖` | 11 | `MeshCore-HowlBot🤖` | No truncation needed |
| `@@MAC` → `A1B2C3D4E5F6` | 12 | `MeshCore-A1B2C3D4E5F6` | MAC resolution, 21 total, no truncation |

### Code location

A static helper function `buildBLEName()` in `src/helpers/nrf52/SerialBLEInterface.cpp` (file-scoped, not in the class). This keeps the logic local to the only place it's needed, and doesn't pollute headers.

```cpp
// Maximum BLE device name length that fits in a scan response AD element.
// 31 (BLE_GAP_ADV_SET_DATA_SIZE_MAX) - 2 (AD length + type bytes) = 29
#define BLE_NAME_MAX_LEN  29

static size_t utf8CharLen(uint8_t lead_byte) {
  if (lead_byte < 0x80) return 1;
  if ((lead_byte & 0xE0) == 0xC0) return 2;
  if ((lead_byte & 0xF0) == 0xE0) return 3;
  if ((lead_byte & 0xF8) == 0xF0) return 4;
  return 1; // invalid lead byte — treat as single byte to avoid infinite loops
}

// Build a BLE device name from prefix + node name, middle-truncating with
// ".." if the result would exceed BLE_NAME_MAX_LEN bytes.
// Truncation is UTF-8 safe and preserves the beginning and end of the name.
static void buildBLEName(char* dest, size_t dest_size,
                         const char* prefix, const char* name)
{
  size_t prefix_len = strlen(prefix);
  size_t name_len = strlen(name);

  // Fast path: fits without truncation
  if (prefix_len + name_len <= BLE_NAME_MAX_LEN) {
    snprintf(dest, dest_size, "%s%s", prefix, name);
    return;
  }

  size_t name_budget = BLE_NAME_MAX_LEN - prefix_len;
  const char sep[] = "..";
  const size_t sep_len = 2;

  // If budget is too small for meaningful middle-truncation (need at least
  // 1 char + sep + 1 char), just take the head
  if (name_budget <= sep_len + 2) {
    memcpy(dest, prefix, prefix_len);
    size_t i = 0;
    while (i < name_budget && i < name_len) {
      size_t cl = utf8CharLen((uint8_t)name[i]);
      if (i + cl > name_budget) break;
      i += cl;
    }
    memcpy(dest + prefix_len, name, i);
    dest[prefix_len + i] = '\0';
    return;
  }

  size_t content_budget = name_budget - sep_len;
  size_t head_target = content_budget / 2;
  size_t tail_target = content_budget - head_target;

  // Walk forward: collect head (complete UTF-8 characters up to head_target bytes)
  size_t head_len = 0;
  {
    size_t i = 0;
    while (i < name_len) {
      size_t cl = utf8CharLen((uint8_t)name[i]);
      if (i + cl > head_target) break;
      i += cl;
    }
    head_len = i;
  }

  // Walk backward: collect tail (complete UTF-8 characters up to tail_target bytes)
  size_t tail_start = name_len;
  size_t tail_len = 0;
  {
    size_t i = name_len;
    while (i > 0 && tail_len < tail_target) {
      // Find start of previous UTF-8 character
      size_t prev = i - 1;
      while (prev > 0 && ((uint8_t)name[prev] & 0xC0) == 0x80)
        prev--;
      size_t cl = i - prev;
      if (tail_len + cl > tail_target) break;
      tail_len += cl;
      i = prev;
    }
    tail_start = name_len - tail_len;
  }

  // Assemble: prefix + head + ".." + tail
  size_t pos = 0;
  memcpy(dest + pos, prefix, prefix_len);     pos += prefix_len;
  memcpy(dest + pos, name, head_len);         pos += head_len;
  memcpy(dest + pos, sep, sep_len);           pos += sep_len;
  memcpy(dest + pos, name + tail_start, tail_len); pos += tail_len;
  dest[pos] = '\0';
}
```

### Changes to `SerialBLEInterface::begin()`

Restore the `main` branch ordering (setName AFTER begin), and use `buildBLEName()`:

```cpp
void SerialBLEInterface::begin(const char* prefix, char* name, uint32_t pin_code) {
  instance = this;

  char charpin[20];
  snprintf(charpin, sizeof(charpin), "%lu", (unsigned long)pin_code);

  Bluefruit.configPrphBandwidth(BANDWIDTH_MAX);
  Bluefruit.begin();

  // Resolve "@@MAC" now that the SoftDevice is active
  if (strcmp(name, "@@MAC") == 0) {
    ble_gap_addr_t addr;
    if (sd_ble_gap_addr_get(&addr) == NRF_SUCCESS) {
      sprintf(name, "%02X%02X%02X%02X%02X%02X",
          addr.addr[5], addr.addr[4], addr.addr[3],
          addr.addr[2], addr.addr[1], addr.addr[0]);
    }
  }

  // Build the BLE name with middle-truncation if needed to fit within
  // the SoftDevice's 31-byte GAP name limit and 29-byte scan response limit
  char dev_name[32+16];
  buildBLEName(dev_name, sizeof(dev_name), prefix, name);
  Bluefruit.setName(dev_name);

  // ... rest of begin() unchanged ...
```

### What is NOT changing

- **Bluefruit library** — no modifications to the third-party library
- **ESP32 BLE interface** — different BLE stack, no SoftDevice constraint
- **NodePrefs / CommonCLI** — no validation changes; the full node name is still stored and used for mesh routing; only the BLE advertised name is truncated
- **Other NRF52 examples** (repeater, room server) — they don't use `SerialBLEInterface`

## TODO

### Phase 1: Implement `buildBLEName()` helper
- [x] Add `BLE_NAME_MAX_LEN` define (29) to `SerialBLEInterface.cpp`
- [x] Add `utf8CharLen()` static function
- [x] Add `buildBLEName()` static function with middle-truncation logic

### Phase 2: Rewrite `begin()` to use `buildBLEName()`
- [x] Restore `main` branch ordering: `Bluefruit.begin()` first, then name setup
- [x] Resolve `@@MAC` immediately after `Bluefruit.begin()`
- [x] Call `buildBLEName()` to produce the truncated BLE name
- [x] Call `Bluefruit.setName()` once, after `begin()`, with the truncated name
- [x] Remove the pre-begin `setName()` call and associated comments

### Phase 3: Verify
- [x] Compile the `t1000e_companion_radio_ble` target successfully
- [x] Trace through algorithm with plan examples to confirm correctness
