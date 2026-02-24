# Research: MeshCore Radio Settings Presets

## Overview

MeshCore radio presets define the LoRa radio parameters for different regions.
They are **not defined in the firmware** — the firmware uses compile-time
defaults (`LORA_FREQ`, `LORA_BW`, `LORA_SF`, `LORA_CR`) and supports runtime
changes via the `set radio <freq>,<bw>,<sf>,<cr>` CLI command.

The presets are defined in **companion apps and tools**:
- **Liam Cottle's official app** (closed-source Flutter, Android/iOS/Web)
- **meshcore-open** (open-source Flutter client by zjs81)
- **config.meshcore.dev** (web-based repeater configuration tool)
- **Ripple GUI** (T-Deck on-device GUI)

Preset changes are community-driven — users propose new presets via GitHub
issues or the MeshCore Discord `#meshcore-app` channel, and Liam Cottle
updates the official app.

## Radio Parameters

Each preset defines five values:

| Parameter | CLI Flag | Range | Description |
|-----------|----------|-------|-------------|
| Frequency | `freq` | 300–2500 MHz | Center frequency in MHz |
| Bandwidth | `bw` | 7.8–500 kHz | LoRa channel bandwidth |
| Spreading Factor | `sf` | 5–12 | Temporal spreading of chirps |
| Coding Rate | `cr` | 5–8 | Forward error correction (4/5 through 4/8) |
| TX Power | `tx` | 1–22 dBm | Transmit power (board-dependent) |

## Current Presets (as of Feb 2026)

Source: [`zjs81/meshcore-open` — `lib/models/radio_settings.dart`](https://github.com/zjs81/meshcore-open/blob/main/lib/models/radio_settings.dart)

Note: The open-source client closely mirrors the official Liam Cottle app
presets. The official app is closed-source, so minor differences may exist (see
"Differences" section below).

### Regional Presets

| Preset Name | Freq (MHz) | BW (kHz) | SF | CR | TX (dBm) |
|---|---|---|---|---|---|
| **Australia** | 915.800 | 250 | 10 | 5 | 20 |
| **Australia (Narrow)** | 916.575 | 62.5 | 7 | 5 | 20 |
| **Australia SA, WA, QLD** | 923.125 | 62.5 | 8 | 5 | 20 |
| **Czech Republic** | 869.432 | 62.5 | 7 | 5 | 14 |
| **EU 433MHz** | 433.650 | 250 | 11 | 5 | 20 |
| **EU/UK (Long Range)** | 869.525 | 250 | 11 | 5 | 14 |
| **EU/UK (Medium Range)** | 869.525 | 250 | 10 | 5 | 14 |
| **EU/UK (Narrow)** | 869.618 | 62.5 | 8 | 5 | 14 |
| **New Zealand** | 917.375 | 250 | 11 | 5 | 20 |
| **New Zealand (Narrow)** | 917.375 | 62.5 | 7 | 5 | 20 |
| **Portugal 433** | 433.375 | 62.5 | 9 | 5 | 20 |
| **Portugal 869** | 869.618 | 62.5 | 7 | 5 | 14 |
| **Switzerland** | 869.618 | 62.5 | 8 | 5 | 14 |
| **USA Arizona** | 908.205 | 62.5 | 10 | 5 | 20 |
| **USA/Canada** | 910.525 | 62.5 | 7 | 5 | 20 |
| **Vietnam** | 920.250 | 250 | 11 | 5 | 20 |

### Off-Grid Presets

These are for private/isolated networks not on a community mesh:

| Preset Name | Freq (MHz) | BW (kHz) | SF | CR | TX (dBm) |
|---|---|---|---|---|---|
| **Off-Grid 433** | 433.000 | 250 | 11 | 5 | 20 |
| **Off-Grid 869** | 869.000 | 250 | 11 | 5 | 14 |
| **Off-Grid 918** | 918.000 | 250 | 11 | 5 | 20 |

## Possible Differences in the Official App

The meshcore-open project is a community-maintained open-source client. The
official Liam Cottle app (closed-source) may include additional or different
presets. Known differences and recent changes from GitHub issues:

- **USA/Canada (Deprecated)** — Removed from official app as of late 2025
  (was 910.525 MHz / SF11 / BW250 / CR5). See [#1070](https://github.com/meshcore-dev/MeshCore/issues/1070).
- **Australia: QLD** — The official app may list QLD separately from SA/WA
  with CR5 (confirmed in [#949](https://github.com/meshcore-dev/MeshCore/issues/949)
  comments: QLD at 923.125 / SF8 / BW62.5 / CR5 vs SA/WA at CR8).
  However, the meshcore-open source shows all three with CR5.
- **Vietnam** — An open issue ([#1804](https://github.com/meshcore-dev/MeshCore/issues/1804))
  requests updating to narrow (920.250 / BW62.5 / SF8 / CR5).

### Pending/Requested Presets (not yet in app)

| Request | Freq (MHz) | BW (kHz) | SF | CR | Issue |
|---|---|---|---|---|---|
| USA: Southern California | 927.875 | 62.5 | 7 | 8 | [#1798](https://github.com/meshcore-dev/MeshCore/issues/1798) |
| Malaysia 919-923 MHz | TBD | TBD | TBD | TBD | [#980](https://github.com/meshcore-dev/MeshCore/issues/980) |
| China 470-510 MHz | 487.875 | 125 | 9 | 5 | [#946](https://github.com/meshcore-dev/MeshCore/issues/946) |
| Vietnam (Narrow) | 920.250 | 62.5 | 8 | 5 | [#1804](https://github.com/meshcore-dev/MeshCore/issues/1804) |

## Historical Context

### Original Settings (pre-2025)

MeshCore started with a single set of radio parameters for all regions:
- SF 10, BW 250, CR 5
- Only the frequency differed by region

### Wide-to-Narrow Migration (2025)

Starting around April 2025, communities began testing narrower bandwidth
settings (BW 62.5 kHz with lower SF). Benefits observed:

- **Lower noise floor**: Narrow band fits between ISM interference (esp. smart
  meters in the US 902-928 MHz band)
- **Better SNR**: Less noise captured in the narrower channel
- **Faster transmissions**: Lower SF = shorter airtime per packet
- **Comparable link budget**: BW62.5/SF7 has ~149 dB link budget vs ~151 dB
  for BW250/SF11 — only ~2 dB difference

The USA/Canada community was one of the first to fully adopt narrow (910.525 /
BW62.5 / SF7 / CR5). The "USA/Canada (Alternate)" preset (the old wide
settings) was deprecated and then removed from the official app in late 2025
([#1070](https://github.com/meshcore-dev/MeshCore/issues/1070)).

Victoria (Australia) also saw dramatic improvements going narrow, with 70-130
km links becoming reliable that didn't work on wide settings.

The EU/UK adopted narrow at 869.618 MHz (offset from the old 869.525 to center
in the 62.5 kHz channel). Poland tested and confirmed these settings worked
well across multiple European countries including the UK, Slovakia, and
Germany.

### Coding Rate Debate

All current presets use CR 4/5 (the minimum). Some communities use higher
coding rates:
- **CR 4/8**: Used by some Australian communities (SA, WA originally used CR8
  per [#949](https://github.com/meshcore-dev/MeshCore/issues/949) but the
  meshcore-open source now shows CR5 for the combined preset)
- **CR 4/8**: Proposed for Southern California ([#1798](https://github.com/meshcore-dev/MeshCore/issues/1798))
- Coding rate can be changed independently — devices with different CR can
  still communicate on the same frequency/BW/SF

### FCC Compliance Discussion

Issue [#945](https://github.com/meshcore-dev/MeshCore/issues/945) raised
that 47 CFR 15.247(a)(2) requires a minimum 6 dB bandwidth of 500 kHz for
DSSS systems in the 902-928 MHz band. This would make BW62.5 technically
non-compliant in the US. The community debated this but ultimately decided
BW500 is impractical due to smart meter interference in most US locations.
This remains an unresolved regulatory question.

## Frequency Band Summary

| Band | Regions | Regulatory Notes |
|---|---|---|
| 433 MHz | EU, Portugal, China | EU: 10 mW ERP typical; China: 50 mW ERP max |
| 868 MHz | EU, UK | 869.4-869.65 sub-band: 500 mW ERP, 10% duty cycle |
| 915 MHz | USA, Canada, Australia, NZ, Vietnam | US: 902-928 MHz ISM; AU/NZ: 915-928 MHz |

### EU 869.4-869.65 MHz Sub-Band

This is the primary MeshCore EU frequency band. Key regulatory advantages:
- 500 mW ERP (higher than the 25 mW allowed on 867.5 MHz)
- 10% duty cycle (vs 2.5% on 867.5 MHz)
- 250 kHz bandwidth permitted
- Same band used by Meshtastic for the same reasons

## Build-Time Defaults in Firmware

The firmware compiles with these defaults (from `platformio.ini`):

```ini
-D LORA_FREQ=869.525
-D LORA_BW=250
-D LORA_SF=11
-D LORA_CR=5
```

These match the **EU/UK (Long Range)** preset. All other regions must change
settings after flashing via CLI (`set radio`) or companion app.

## Sources

- [MeshCore FAQ — docs/faq.md](https://github.com/meshcore-dev/MeshCore/blob/main/docs/faq.md)
- [meshcore-open radio_settings.dart](https://github.com/zjs81/meshcore-open/blob/main/lib/models/radio_settings.dart)
- [RFC: Removal of USA/Canada (Alternate) #1070](https://github.com/meshcore-dev/MeshCore/issues/1070)
- [Australia: Perth preset #949](https://github.com/meshcore-dev/MeshCore/issues/949)
- [Australia: NSW preset #1030](https://github.com/meshcore-dev/MeshCore/issues/1030)
- [Vietnam preset change #860](https://github.com/meshcore-dev/MeshCore/issues/860)
- [Vietnam narrow request #1804](https://github.com/meshcore-dev/MeshCore/issues/1804)
- [USA Regulatory preset suggestion #945](https://github.com/meshcore-dev/MeshCore/issues/945)
- [Geographical Presets & Coding Rate #549](https://github.com/meshcore-dev/MeshCore/issues/549)
- [Southern California preset #1798](https://github.com/meshcore-dev/MeshCore/issues/1798)
- [China 470 MHz preset #946](https://github.com/meshcore-dev/MeshCore/issues/946)
- [Malaysia presets #980](https://github.com/meshcore-dev/MeshCore/issues/980)
- [MeshCore Switzerland settings](https://www.meshcore.ch/settings/)
- [Poland presets](https://lorameshcore.pl/aktualnePresetyPolska/)
- [MeshCore CLI commands — docs/cli_commands.md](https://github.com/meshcore-dev/MeshCore/blob/main/docs/cli_commands.md)
- [config.meshcore.dev](https://config.meshcore.dev/)
- [LoRa/MeshCore analysis — housedillon.com](https://housedillon.com/blog/meshcore-and-lorawan/)
