# Plan: GitHub Issue/PR/Discussion Templates for MeshCore

**Date:** 2026-02-24
**Research:** [research-github-issue-templates.md](research-github-issue-templates.md)

---

## PR Strategy

Five separate PRs, ordered by impact. Each targets `upstream/dev`.

| PR | Files | Why grouped |
|----|-------|-------------|
| 1. Bug Report + config.yml | `bug_report.yml`, `config.yml` | config.yml sets up contact links and chooser; `blank_issues_enabled: true` initially |
| 2. PR Template | `pull_request_template.md` | Independent |
| 3. New Hardware Request | `hardware.yml` | Independent |
| 4. Build Issue | `build_issue.yml` | Independent |
| 5. Discussion Templates + disable blank issues | `ideas.yml`, `q-a.yml`, update `config.yml` | Final PR flips `blank_issues_enabled: false` once all templates are in place |

---

## PR 1: Bug Report Template + Issue Chooser Config

### Files

- `.github/ISSUE_TEMPLATE/bug_report.yml`
- `.github/ISSUE_TEMPLATE/config.yml`

### `config.yml`

```yaml
blank_issues_enabled: true
contact_links:
  - name: Ask a Question
    url: https://github.com/meshcore-dev/MeshCore/discussions/categories/q-a
    about: Use GitHub Discussions for questions and support.
  - name: Feature Request / Idea
    url: https://github.com/meshcore-dev/MeshCore/discussions/categories/ideas
    about: Suggest features and improvements in Discussions.
  - name: Discord Community
    url: https://discord.gg/BMwCtwHj5V
    about: Join the Discord for real-time help and chat.
```

**Notes:**
- `blank_issues_enabled: true` initially — blank issues remain available as a
  fallback while the other issue templates (hardware request, build issue) are
  not yet merged. The final PR (PR 5) flips this to `false` once all templates
  are in place.
- Feature requests routed to Discussions/Ideas (keeps tracker focused on bugs)
- Support questions routed to Discussions/Q&A and Discord
- Discord link taken from upstream README

### `bug_report.yml`

```yaml
name: Bug Report
description: Report a bug in MeshCore firmware or applications.
title: "[Bug]: "
labels: ["bug"]
body:
  - type: checkboxes
    id: prerequisites
    attributes:
      label: Prerequisites
      description: Please confirm the following before submitting.
      options:
        - label: I have searched [existing issues](https://github.com/meshcore-dev/MeshCore/issues) for duplicates.
          required: true

  - type: dropdown
    id: board
    attributes:
      label: Board / Hardware
      description: Which board/hardware are you using?
      options:
        - Seeed T1000-E (Card Tracker)
        - Seeed SenseCAP Solar Node
        - Seeed Wio Tracker L1
        - LilyGo T-Echo
        - LilyGo T-Echo Lite
        - LilyGo T-Deck
        - LilyGo T-Beam Supreme
        - LilyGo T-Beam-1W
        - LilyGo T3-S3
        - LilyGo TLora V2.1-1.6
        - LilyGo TLora C6
        - RAK4631
        - RAK3172
        - RAK3401
        - Heltec LoRa32 v2
        - Heltec LoRa32 v3
        - Heltec LoRa32 v4
        - Heltec Mesh Pocket
        - Heltec Mesh Solar
        - Heltec T114
        - Heltec Wireless Paper
        - Heltec Wireless Tracker
        - Heltec Tracker v2
        - Heltec Vision Master E213
        - Heltec Vision Master E290
        - Heltec Vision Master T190
        - Heltec CT62
        - ThinkNode M1 (ELECROW eink)
        - ThinkNode M3 (ELECROW nrf)
        - ThinkNode M6 (ELECROW solar)
        - Station G2
        - Ikoka Handheld
        - Ikoka Nano
        - Ikoka Stick
        - WHY2025 Badge
        - Meshtiny
        - Xiao nRF52840
        - Xiao ESP32-S3
        - Xiao ESP32-C3
        - Xiao ESP32-C6
        - Xiao RP2040
        - M5Stack Unit C6L
        - ProMicro nRF52840
        - Waveshare RP2040 LoRa
        - Raspberry Pi Pico W
        - Other (describe in Additional Context)
    validations:
      required: true

  - type: input
    id: firmware-version
    attributes:
      label: Firmware Version
      description: >-
        Exact version string from the companion app device info or CLI 'version'
        command (e.g. "1.5.2" or "1.5.2-abc1234").
      placeholder: "e.g., 1.5.2-abc1234"
    validations:
      required: true

  - type: dropdown
    id: firmware-type
    attributes:
      label: Firmware Type
      description: Which firmware variant are you running?
      options:
        - Companion Radio (BLE)
        - Companion Radio (USB)
        - Companion Radio (WiFi)
        - Simple Repeater
        - Room Server
        - KISS Protocol
        - Other
    validations:
      required: true

  - type: dropdown
    id: category
    attributes:
      label: Category
      description: What area is affected? Select all that apply.
      multiple: true
      options:
        - BLE
        - LoRa / Mesh Routing
        - Serial / USB
        - WiFi
        - Display / UI
        - GPS
        - Sensors
        - Power Management
        - Other
    validations:
      required: true

  - type: textarea
    id: description
    attributes:
      label: Bug Description
      description: What happened? What did you expect to happen instead?
    validations:
      required: true

  - type: textarea
    id: reproduction
    attributes:
      label: Steps to Reproduce
      description: Detailed steps to reproduce the behavior.
      placeholder: |
        1. Flash firmware version X on board Y
        2. Pair with companion app
        3. Send a message to...
        4. Observe that...
    validations:
      required: true

  - type: textarea
    id: logs
    attributes:
      label: Relevant Log Output
      description: >-
        Paste serial/USB log output, BLE debug output, or crash traces.
        Leave blank if not applicable.
      render: shell

  - type: input
    id: companion-app
    attributes:
      label: Companion App & Version
      description: >-
        If relevant, which companion app and version?
        (e.g., "MeshCore Android 2.1.0", "MeshCore iOS 1.3.0")
      placeholder: "e.g., MeshCore Android 2.1.0"

  - type: textarea
    id: additional
    attributes:
      label: Additional Context
      description: >-
        Screenshots, radio configuration, frequency/region preset, mesh
        topology, or anything else relevant.
```

**Design decisions:**
- Only `bug` label auto-applied (exists upstream). No `triage` label — doesn't exist.
- Board dropdown: 45 boards covering all platforms, alphabetized by manufacturer.
  "Other" option at the end. This is the most maintenance-heavy field but also
  the highest-value for triage.
- Platform as separate field from board — useful for filtering, and some boards
  have multiple SoC variants.
- Firmware type field — distinguishes companion radio (BLE/USB/WiFi), repeater,
  room server behavior. Critical for reproducing issues.
- Category multi-select — bugs often span multiple areas (e.g., BLE + Power).
- `render: shell` on logs — prevents Markdown mangling.
- Prerequisite checkboxes — "searched existing issues" is required.
- Companion app field is optional — not all bugs involve the app.

---

## PR 2: Pull Request Template

### File

- `.github/pull_request_template.md`

```markdown
## Summary

<!-- What does this PR do and why? Link to related issues with "Fixes #123". -->

## Type of Change

- [ ] Bug fix
- [ ] New feature
- [ ] Board / variant support
- [ ] Breaking change
- [ ] Build system / CI
- [ ] Refactor / code quality
- [ ] Documentation

## Hardware Tested

<!-- List the hardware you tested on, or "None" if you don't have hardware. -->
<!-- e.g., T1000-E, Heltec LoRa32 v3, RAK4631, T-Deck -->

## Test Plan

<!-- How did you verify this change works? -->

- [ ] Compiles without warnings on target environment(s): ___
- [ ] Tested on physical hardware
- [ ] Tested via companion app: ___
```

**Design decisions:**
- Minimal sections — Summary, Type, Hardware Tested, Test Plan. No checklist
  bloat.
- Hardware tested is free-text with a suggested format — too many hardware
  options for checkboxes, and contributors know what they tested on.
- No "I followed coding conventions" checkbox — that's what review is for.
- No "docs updated" checkbox — most firmware PRs don't need doc changes.
- Compact enough that contributors won't be tempted to delete the whole thing.

---

## PR 3: New Hardware Request Template

### File

- `.github/ISSUE_TEMPLATE/hardware.yml`

```yaml
name: New Board/Hardware Request
description: Request support for a new board/hardware.
title: "[Hardware]: "
labels: ["enhancement"]
body:
  - type: checkboxes
    id: prerequisites
    attributes:
      label: Prerequisites
      options:
        - label: I have searched [existing issues](https://github.com/meshcore-dev/MeshCore/issues?q=label%3Aenhancement) for this hardware.
          required: true

  - type: dropdown
    id: soc
    attributes:
      label: SoC / Platform
      description: Which system-on-chip does this board/hardware use?
      multiple: true
      options:
        - NRF52840
        - ESP32
        - ESP32-S3
        - ESP32-C3
        - ESP32-C6
        - RP2040
        - STM32WLE5
        - Other (describe below)
    validations:
      required: true

  - type: input
    id: lora-ic
    attributes:
      label: LoRa IC
      description: Which LoRa radio IC does this board/hardware use?
      placeholder: "e.g., SX1262, SX1276, LR1110, LR1121"
    validations:
      required: true

  - type: input
    id: product-link
    attributes:
      label: Product Link
      description: URL to the product page, store listing, or datasheet.
      placeholder: "https://..."
    validations:
      required: true

  - type: textarea
    id: description
    attributes:
      label: board/hardware Description
      description: >-
        Describe the board/hardware. Include pinout, schematic links, available
        peripherals (display, GPS, sensors), and any other relevant details.
    validations:
      required: true
```

**Design decisions:**
- Similar to Meshtastic's `New Board.yml` — proven pattern for multi-hardware
  firmware projects.
- SoC multi-select (some boards have multiple radios or variants).
- LoRa IC is required — essential for driver selection.
- Product link is required — maintainers need to see documentation.
- `enhancement` label auto-applied (exists upstream).

---

## PR 4: Build Issue Template

### File

- `.github/ISSUE_TEMPLATE/build_issue.yml`

```yaml
name: Build Issue
description: Report a compilation or build system problem.
title: "[Build]: "
labels: ["bug"]
body:
  - type: checkboxes
    id: prerequisites
    attributes:
      label: Prerequisites
      options:
        - label: I have searched [existing issues](https://github.com/meshcore-dev/MeshCore/issues) for this build error.
          required: true
        - label: I am building from the latest `dev` branch.

  - type: input
    id: build-target
    attributes:
      label: Build Target
      description: >-
        PlatformIO environment name from platformio.ini
        (e.g., "t1000e_companion_radio_ble").
      placeholder: "e.g., t1000e_companion_radio_ble"
    validations:
      required: true

  - type: dropdown
    id: host-os
    attributes:
      label: Host Operating System
      options:
        - Linux
        - macOS
        - Windows
        - Devcontainer / Docker
        - Other
    validations:
      required: true

  - type: input
    id: pio-version
    attributes:
      label: PlatformIO Version
      description: "Run `pio --version` to find this."
      placeholder: "e.g., 6.1.16"
    validations:
      required: true

  - type: textarea
    id: error-output
    attributes:
      label: Error Output
      description: Paste the build error output.
      render: shell
    validations:
      required: true

  - type: textarea
    id: additional
    attributes:
      label: Additional Context
      description: >-
        Any modifications to platformio.ini, custom board definitions, toolchain
        version overrides, or other relevant details.
```

**Design decisions:**
- Separates build/toolchain issues from runtime firmware bugs — different
  information is needed.
- Build target field asks for the PlatformIO environment name (actionable for
  maintainers).
- PlatformIO version is required — many build issues are version-specific.
- `render: shell` on error output.
- `bug` label auto-applied.

---

## PR 5: Discussion Templates + Disable Blank Issues

### Files

- `.github/DISCUSSION_TEMPLATE/ideas.yml`
- `.github/DISCUSSION_TEMPLATE/q-a.yml`
- `.github/ISSUE_TEMPLATE/config.yml` (update `blank_issues_enabled` to `false`)

### `ideas.yml`

```yaml
title: "[Idea]: "
labels: ["enhancement"]
body:
  - type: input
    id: hardware
    attributes:
      label: Hardware (if applicable)
      description: Which hardware does this apply to? Leave blank if not hardware-specific.
      placeholder: "e.g., T1000-E, Heltec LoRa32 v3, all ESP32 boards"

  - type: textarea
    id: description
    attributes:
      label: Describe your idea
      description: What would you like to see? What problem does it solve?
    validations:
      required: true

  - type: textarea
    id: alternatives
    attributes:
      label: Alternatives considered
      description: Have you considered any alternative solutions or workarounds?

  - type: textarea
    id: additional
    attributes:
      label: Additional context
      description: Any other context, mockups, or references.
```

### `q-a.yml`

```yaml
title: "[Q&A]: "
body:
  - type: dropdown
    id: topic
    attributes:
      label: Topic Area
      options:
        - Getting Started / Setup
        - BLE / Companion App
        - LoRa / Mesh Networking
        - Build System / Compilation
        - Hardware / Wiring
        - Other
    validations:
      required: true

  - type: input
    id: hardware
    attributes:
      label: Hardware (if applicable)
      placeholder: "e.g., T1000-E, RAK4631"

  - type: textarea
    id: question
    attributes:
      label: Your Question
      description: Describe what you need help with.
    validations:
      required: true

  - type: textarea
    id: context
    attributes:
      label: What have you tried?
      description: Steps you have already taken or documentation you have read.
```

**Design decisions:**
- Filenames match the existing Discussion category slugs: `ideas`, `q-a`.
- Ideas template uses a hardware input field — people think in terms of
  hardware names, not chip families. Optional so non-hardware ideas aren't blocked.
- Q&A template is minimal — low barrier to asking questions.
- `enhancement` label auto-applied on Ideas (exists upstream).
- No label on Q&A — questions aren't enhancements or bugs.
- This final PR flips `blank_issues_enabled` from `true` to `false` in
  `config.yml`, now that all issue templates (bug report, hardware request,
  build issue) are in place.
