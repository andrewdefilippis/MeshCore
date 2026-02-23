# MeshCore Project

## Build System
- PlatformIO (`pio` binary at `/usr/local/py-utils/bin/pio`)
- Build a target: `pio run -e <environment_name>`
- Environment names are defined in `platformio.ini` and `variants/*/platformio.ini`
- Common T1000-E target: `t1000e_companion_radio_ble`

## Architecture
- Embedded C++ targeting NRF52840 (Nordic) and ESP32 platforms
- Nordic builds use Adafruit Bluefruit nRF52 framework with SoftDevice S140 v7.3.0
- Bluefruit library lives in `~/.platformio/packages/framework-arduinoadafruitnrf52/libraries/Bluefruit52Lib/`
- The Bluefruit library is patched at build time (see build scripts) — do NOT modify it directly
- ESP32 builds use the ESP-IDF BLE stack (NimBLE)

## Key Directories
- `src/helpers/` — shared helper code (BLE interfaces, CLI, UI, mesh helpers)
  - `src/helpers/nrf52/` — NRF52-specific implementations
  - `src/helpers/esp32/` — ESP32-specific implementations
- `examples/companion_radio/` — companion radio app (BLE pairing with phone)
- `variants/` — board-specific configurations and sensor code
- `boards/` — PlatformIO board definition JSON files
- `lib/` — vendored libraries (including NRF52 SoftDevice headers)

## Platform Differences
- NRF52 and ESP32 have separate `SerialBLEInterface` implementations with the same API
- NRF52 BLE has a 31-byte default GAP device name limit (`BLE_GAP_DEVNAME_DEFAULT_LEN`) — the Bluefruit library does not configure `BLE_GAP_CFG_DEVICE_NAME`
- `Bluefruit::setName()` does not check the return value of `sd_ble_gap_device_name_set()` — failures are silent
- ESP32 BLE stack does not have the same name length constraint

## Conventions
- Node names are stored in `NodePrefs.node_name[32]` (31 chars + NUL)
- BLE device names are prefixed with `BLE_NAME_PREFIX` ("MeshCore-", 9 bytes)
- `ADVERT_NAME` is a compile-time default for node_name; "@@MAC" is a special value resolved at runtime
- Board fallback names come from `usb_product` in `boards/*.json` (e.g. "T1000-E-BOOT")

## Testing
- No unit test framework — verify by compilation and algorithm tracing
- Build the relevant PlatformIO environment to confirm changes compile
- For NRF52 BLE changes, the primary test target is `t1000e_companion_radio_ble`

## Git
- Fork remote: `origin` (andrewdefilippis/MeshCore)
- Upstream remote: `upstream` (meshcore-dev/MeshCore)
- PRs go against `dev` branch on upstream
- GPG signing required for commits
