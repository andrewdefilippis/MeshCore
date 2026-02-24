# Examples: GitHub Template Rendered Output

These examples show how each template looks after a user fills it out and
submits. GitHub YAML form fields render as Markdown with `### Field Label`
headings followed by the user's response.

---

## 1. Bug Report (bug_report.yml)

> **Title:** [Bug]: BLE disconnects after 30 seconds when node name exceeds 22 characters
>
> **Labels:** `bug`

### Prerequisites

- [X] I have searched [existing issues](https://github.com/meshcore-dev/MeshCore/issues) for duplicates.

### Board / Hardware

Seeed T1000-E (Card Tracker)

### Firmware Version

1.5.2-a3f8c01

### Firmware Type

Companion Radio (BLE)

### Category

BLE, LoRa / Mesh Routing

### Bug Description

When I set a node name longer than 22 characters (e.g. "MyLongNodeNameForTesting"), the BLE connection drops approximately 30 seconds after pairing with the MeshCore Android app. Shorter names (22 characters or fewer) work fine.

The device shows as connected briefly in the app, then disconnects. The serial log shows the GAP device name was set successfully but the advertising data appears truncated.

Expected behavior: BLE connection should remain stable regardless of node name length within the 31-character NodePrefs limit.

### Steps to Reproduce

1. Flash firmware 1.5.2-a3f8c01 on T1000-E with `t1000e_companion_radio_ble` environment
2. Set node name to "MyLongNodeNameForTesting" (24 characters) via companion app
3. Reboot the device
4. Open MeshCore Android app and pair with the device
5. Wait 30 seconds — BLE connection drops
6. Repeat with a 20-character name — connection remains stable

### Relevant Log Output

```shell
[BLE] GAP device name set: MeshCore-MyLongNodeNam (truncated)
[BLE] Advertising started
[BLE] Central connected: AA:BB:CC:DD:EE:FF
[BLE] MTU negotiated: 247
[BLE] Connection established
...
[BLE] Disconnect event: reason=0x13 (Remote User Terminated)
```

### Companion App & Version

MeshCore Android 2.1.0

### Additional Context

I tested this on two different T1000-E units with the same result. The issue does not occur on my Heltec LoRa32 v3 (ESP32-S3) running the same firmware version, which suggests it's NRF52-specific.

Region preset: US 915 Long Fast

---

## 2. Bug Report — Minimal (same template, fewer optional fields)

> **Title:** [Bug]: GPS not getting a fix on T-Echo after firmware update
>
> **Labels:** `bug`

### Prerequisites

- [X] I have searched [existing issues](https://github.com/meshcore-dev/MeshCore/issues) for duplicates.

### Board / Hardware

LilyGo T-Echo

### Firmware Version

1.5.1

### Firmware Type

Companion Radio (BLE)

### Category

GPS

### Bug Description

After updating from 1.4.8 to 1.5.1, the GPS on my T-Echo never gets a fix. It worked fine on the previous firmware. I've waited over 10 minutes outdoors with clear sky view.

### Steps to Reproduce

1. Flash 1.5.1 on T-Echo
2. Take device outdoors with clear sky view
3. Wait for GPS fix — it never acquires

### Relevant Log Output

_No response_

### Companion App & Version

_No response_

### Additional Context

_No response_

---

## 3. Pull Request Template (pull_request_template.md)

> **Title:** Fix BLE GAP device name length handling on NRF52

## Summary

Fixes silent truncation of BLE device names longer than 22 characters on NRF52
platforms. The SoftDevice's default `BLE_GAP_DEVNAME_DEFAULT_LEN` (31 bytes)
was insufficient for the 9-byte "MeshCore-" prefix plus a 31-character node
name. This configures the GAP device name length via `BLE_GAP_CFG_DEVICE_NAME`
in the SoftDevice configuration and adds a compile-time check.

Fixes #1769

## Type of Change

- [x] Bug fix
- [ ] New feature
- [ ] Board / variant support
- [ ] Breaking change
- [ ] Build system / CI
- [ ] Refactor / code quality
- [ ] Documentation

## Hardware Tested

T1000-E, RAK4631

## Test Plan

- [x] Compiles without warnings on target environment(s): `t1000e_companion_radio_ble`, `rak4631_companion_radio_ble`
- [x] Tested on physical hardware
- [x] Tested via companion app: MeshCore Android 2.1.0

---

## 4. Pull Request — Minimal (contributor without hardware)

> **Title:** Add Station G2 variant definition

## Summary

Adds board definition and variant configuration for the BQ Station G2
(ESP32-S3 + SX1262). Pin mappings sourced from the manufacturer's schematic
(linked in variant README).

Relates to #1790

## Type of Change

- [ ] Bug fix
- [ ] New feature
- [x] Board / variant support
- [ ] Breaking change
- [ ] Build system / CI
- [ ] Refactor / code quality
- [ ] Documentation

## Hardware Tested

None — I do not have this hardware. Pin mappings are from the datasheet.

## Test Plan

- [x] Compiles without warnings on target environment(s): `station_g2_companion_radio_ble`
- [ ] Tested on physical hardware
- [ ] Tested via companion app: ___

---

## 5. New Hardware Request (hardware.yml)

> **Title:** [Hardware]: Ebyte EoRa-S3 900TB
>
> **Labels:** `enhancement`

### Prerequisites

- [X] I have searched [existing issues](https://github.com/meshcore-dev/MeshCore/issues?q=label%3Aenhancement) for this hardware.

### SoC / Platform

ESP32-S3

### LoRa IC

SX1262

### Product Link

https://www.cdebyte.com/products/EoRa-S3-900TB

### Board/Hardware Description

The Ebyte EoRa-S3 is a compact module with ESP32-S3 + SX1262. It has a U.FL antenna connector and exposes UART, SPI, I2C, and GPIO pins on castellated pads.

Key specs:
- ESP32-S3-WROOM-1 (8MB Flash, 2MB PSRAM)
- SX1262 LoRa transceiver (868/915 MHz)
- 22 dBm max TX power
- USB-C for programming
- No onboard display, GPS, or sensors

Schematic: https://www.cdebyte.com/products/EoRa-S3-900TB#downloads

Pin mapping (from datasheet):
- LoRa NSS: GPIO 10
- LoRa SCK: GPIO 12
- LoRa MOSI: GPIO 11
- LoRa MISO: GPIO 13
- LoRa DIO1: GPIO 14
- LoRa RESET: GPIO 5
- LoRa BUSY: GPIO 4

---

## 6. Build Issue (build_issue.yml)

> **Title:** [Build]: Compilation fails on t1000e_companion_radio_ble with GCC 14
>
> **Labels:** `bug`

### Prerequisites

- [X] I have searched [existing issues](https://github.com/meshcore-dev/MeshCore/issues) for this build error.
- [X] I am building from the latest `dev` branch.

### Build Target

t1000e_companion_radio_ble

### Host Operating System

Linux

### PlatformIO Version

6.1.16

### Error Output

```shell
variants/t1000-e/t1000e_sensors.cpp:42:5: error: narrowing conversion of '-40' from 'int' to 'char' [-Wnarrowing]
   42 |     -40, -39, -38, -37, -36, -35, -34, -33, -32, -31,
      |     ^~~
variants/t1000-e/t1000e_sensors.cpp:42:10: error: narrowing conversion of '-39' from 'int' to 'char' [-Wnarrowing]
   42 |     -40, -39, -38, -37, -36, -35, -34, -33, -32, -31,
      |          ^~~
*** [.pio/build/t1000e_companion_radio_ble/src/variants/t1000-e/t1000e_sensors.cpp.o] Error 1
```

### Additional Context

PlatformIO installed `toolchain-gccarmnoneeabi@1.140201.0` (GCC 14.2). The `nordicnrf52` platform's default toolchain range (`>=1.60301.0,<1.80000.0`) doesn't include this version, so I pinned it in `platformio.ini`. The narrowing conversion error only appears with GCC 14 — GCC 12 accepted `char` arrays with negative initializers.

---

## 7. Discussion — Ideas (ideas.yml)

> **Title:** [Idea]: Show packet error rate as a percentage in the companion app
>
> **Labels:** `enhancement`
>
> **Category:** Ideas

### Hardware (if applicable)

_No response_

### Describe your idea

Currently the companion app shows raw packet counts (sent/received/failed) but doesn't calculate a packet error rate. It would be useful to see a percentage like "3.2% packet loss" alongside the raw counts, especially when comparing different radio configurations or antenna placements.

This could be calculated from the existing counters — no firmware change needed, just a UI addition in the companion apps.

### Alternatives considered

I've been manually calculating this from the raw numbers, but it's tedious when comparing multiple configurations. A spreadsheet works but having it in the app would be much more convenient.

### Additional context

Meshtastic shows a similar metric in their device info panel. Screenshot for reference:

[screenshot would be attached here]

---

## 8. Discussion — Q&A (q-a.yml)

> **Title:** [Q&A]: How to set up a repeater on RAK4631?
>
> **Category:** Q&A

### Topic Area

Getting Started / Setup

### Hardware (if applicable)

RAK4631

### Your Question

I have a RAK4631 WisBlock and want to set it up as a simple repeater to extend my mesh network. I've flashed the `rak4631_simple_repeater` firmware successfully, but the device doesn't seem to be forwarding messages between my two companion radio nodes.

The serial output shows it booting fine, but I never see any "relay" or "forward" messages in the log. Do I need to configure anything after flashing, or should it work out of the box?

### What have you tried?

- Flashed `rak4631_simple_repeater` firmware via PlatformIO
- Confirmed the device boots (serial output shows version and "Ready")
- Placed the repeater between two companion radio nodes (~500m apart)
- Sent messages between the two companion nodes — they only arrive when in direct range, not through the repeater
- Checked that all three devices are on the same frequency preset (US 915 Long Fast)
- Read the simple_repeater example README but it doesn't mention any post-flash configuration

---

## 9. Issue Chooser (config.yml)

When a user clicks "New Issue", they see a chooser page with:

| Option | Type | Destination |
|--------|------|-------------|
| **Bug Report** | Template | `bug_report.yml` form |
| **New Board/Hardware Request** | Template | `hardware.yml` form |
| **Build Issue** | Template | `build_issue.yml` form |
| **Ask a Question** | External link | Discussions > Q&A |
| **Feature Request / Idea** | External link | Discussions > Ideas |
| **Discord Community** | External link | discord.gg/BMwCtwHj5V |

Blank issues are **disabled** — users must pick one of the above.
