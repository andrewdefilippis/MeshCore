# Research: GitHub Issue & PR Templates from Similar Projects

**Date:** 2026-02-23
**Purpose:** Survey issue/PR template patterns from embedded/mesh/radio projects to inform MeshCore's own template design.

---

## Current MeshCore State

MeshCore upstream (`meshcore-dev/MeshCore`) has a `.github/` directory containing only `actions/` and `workflows/` -- no issue templates, no PR template, no config.yml.

---

## 1. Meshtastic/firmware (LoRa mesh networking firmware)

**The closest comparable project.** Uses YAML form templates exclusively for issues.

### Issue Templates

| File | Format | Title Prefix |
|------|--------|-------------|
| `Bug Report.yml` | YAML form | `[Bug]: ` |
| `New Board.yml` | YAML form | `[Board]: ` |
| `feature.yml` | YAML form | `[Feature Request]: ` |

#### Bug Report (`Bug Report.yml`)
- **Labels auto-applied:** `bug`, `triage`
- **Fields collected:**
  - **Category** (required dropdown, multi-select): Hardware Compatibility, BLE, Serial, WiFi, Other
  - **Hardware** (required dropdown, multi-select): ~40 specific boards listed (T-Beam, T-Echo, Rak4631, Heltec V3, Seeed Card Tracker T1000-E, etc.) plus DIY and Other
  - **UI component checkboxes**: MUI colorTFT, InkHUD ePaper, OLED slide UI
  - **Firmware Version** (required input): placeholder `x.x.x.yyyyyyy`
  - **Description** (required textarea)
  - **Relevant log output** (optional textarea, rendered as Shell)
- **Notable patterns:**
  - Exhaustive hardware dropdown avoids free-text ambiguity
  - Multi-select allows reporting bugs that span multiple boards
  - Log output uses `render: Shell` for automatic syntax highlighting
  - Title prefix `[Bug]: ` enables quick scanning

#### New Board (`New Board.yml`)
- **Labels:** `enhancement`, `triage`
- **Fields:**
  - **SOC** (required dropdown, multi-select): NRF52, ESP32, Other
  - **LoRa IC** (required input)
  - **Product Link** (required input)
  - **Description** (required textarea)
- **Notable:** Domain-specific template for hardware support requests -- separates "new board" from generic feature requests

#### Feature Request (`feature.yml`)
- **Labels:** `enhancement`
- **Fields:**
  - **Platform** (required dropdown, multi-select): NRF52, ESP32, RP2040, Linux Native, Cross-Platform, Other
  - **Description** (required textarea)
- **Minimal and focused.** Platform dropdown is the only structured field.

### PR Template (`pull_request_template.md`)
- Markdown format
- Opens with tips section clearly marked for deletion: "Please delete all these tips"
- Tips include:
  - Open an issue first for large changes
  - Don't check in unchanged files
  - Don't reformat lines you didn't change
  - Recommends VS Code + Trunk Check for consistent formatting
  - Mention "fixes #bugnum" for bug fix PRs
  - Enable "Allow edits by maintainers"
  - Don't submit untested code
  - If you don't have hardware, say so
- **Attestations checklist:**
  - [ ] Tested that changes behave as described
  - [ ] Tested no obvious regressions on:
    - [ ] Heltec V3
    - [ ] T-Deck
    - [ ] T-Beam
    - [ ] RAK WisBlock 4631
    - [ ] Seeed T-1000E
    - [ ] Other (please specify)
- **Notable patterns:**
  - Hardware-specific regression checklist is very useful for multi-platform firmware
  - Explicit "I don't have this hardware" escape hatch
  - Contributor Discord role incentive mentioned

### Other Files
- `FUNDING.yml` -- OpenCollective link
- `copilot-instructions.md` -- AI assistant context (hardware platforms, radio chips, architecture)
- `meshtastic_logo.png` -- Branding asset for issue template display

---

## 2. RFQuack/RFQuack (RF security research tool)

**Smaller project, uses older markdown templates.**

### Issue Templates

| File | Format | Title Prefix |
|------|--------|-------------|
| `bug_report.md` | Markdown | (none) |
| `feature_request.md` | Markdown | (none) |
| `vulnerability.md` | Markdown | `[VULN] ` |

#### Bug Report (`bug_report.md`)
- **Labels:** `Bug`
- **Sections:**
  - Describe the bug (with links to verbose logging and RadioLib debug mode)
  - To Reproduce (minimal build steps or PlatformIO project)
  - Expected behavior
  - Screenshots or pictures (explicitly mentions wiring photos)
  - Additional info:
    - MCU: [e.g., ESP32, ESP8266]
    - Wireless module type [e.g., CC1101, SX1268]
    - Host environment [e.g., Docker, Linux, macOS, Windows]
    - RFQuack version, branch, or tag
    - RadioLib version, branch, or tag

#### Feature Request (`feature_request.md`)
- **Labels:** `Feature request`
- **Note:** This template is actually a copy-paste of the bug report template (appears to be an error -- it asks for "Describe the bug" in a feature request)

#### Vulnerability (`vulnerability.md`)
- **Labels:** `Vulnerability`
- **Title prefix:** `[VULN] `
- **Unique template type** -- collects:
  - High-level overview of vulnerability and possible effect
  - Detailed description (code line, code path)
  - How the vulnerability was found (tools/procedure)
  - Exploitability assessment
  - Root cause identification
  - Version information and hardware requirements
- **Notable:** Having a dedicated vulnerability report template is good security hygiene for RF projects

#### No PR template, no config.yml

---

## 3. jgromes/RadioLib (multi-platform radio library)

**Uses markdown templates with a config.yml that disables blank issues.**

### Issue Templates

| File | Format | Title Prefix |
|------|--------|-------------|
| `bug_report.md` | Markdown | (none) |
| `feature_request.md` | Markdown | (none) |
| `module-not-working.md` | Markdown | (none) |
| `regular-issue.md` | Markdown | (none) |
| `config.yml` | YAML config | N/A |

#### Bug Report (`bug_report.md`)
- **Fields:**
  - Describe the bug (with link to appropriate debug mode)
  - Debug mode output (in collapsible `<details>` block)
  - To Reproduce: "Minimal Arduino sketch" (in collapsible `<details>` block with `cpp` syntax highlighting)
  - Expected behavior
  - Screenshots
  - Additional info:
    - MCU
    - Link to Arduino core
    - Wireless module type
    - Arduino IDE version
    - Library version (or git hash)
- **Notable patterns:**
  - Uses `<details><summary>` HTML for collapsible sections (debug output, sketch code)
  - Explicitly links to troubleshooting guide, API docs, and online status code decoder
  - Asks for link to Arduino core (not just "ESP32" but the actual core repo URL)

#### Module Not Working (`module-not-working.md`)
- **Separate template for "my module doesn't work"** -- distinct from software bugs
- **Pre-submission checklist** (5 points):
  1. Read CONTRIBUTING.md (warns: issues that don't follow will be closed/locked/deleted)
  2. Check Troubleshooting Guide, API docs, status code decoder
  3. Use latest release
  4. Use Arduino forums for generic questions
  5. Check error codes page
- **Fields:**
  - Sketch (collapsible)
  - Hardware setup: wiring diagram, schematic, pictures
  - Debug mode output (collapsible)
  - Additional info (same as bug report)
- **Notable:** Separating "hardware not working" from "software bug" reduces noise

#### Feature Request (`feature_request.md`)
- Standard format: problem description, proposed solution, alternatives, additional context

#### Regular Issue (`regular-issue.md`)
- Catch-all template with only the pre-submission checklist
- **Minimal body** -- just the 5-point checklist

#### config.yml
```yaml
blank_issues_enabled: false
contact_links:
  - name: RadioLib Discussions
    url: https://github.com/jgromes/RadioLib/discussions
    about: Please ask generic questions here.
```
- **Disables blank issues** -- forces template selection
- **Redirects generic questions** to GitHub Discussions

### PR Template (`pull_request_template.md`)
- States 4 rules:
  1. Code must be tested, impacts understood
  2. All CI must pass (lists: Arduino compilation, ESP-IDF, RPi, runtime test, CodeQL, Cppcheck)
  3. Follow code style in CONTRIBUTING.md
  4. PR review is constructive, expect feedback
- Instructs contributor to delete template and replace with explanation of goal, rationale, and impacts
- **Compact and clear.** No checkboxes -- trusts that CI enforces quality.

---

## 4. Adafruit/Adafruit_nRF52_Arduino (NRF52 Arduino framework)

**Mixed approach: YAML form for bug reports, markdown for feature requests.**

### Issue Templates

| File | Format | Title Prefix |
|------|--------|-------------|
| `bug_report.yml` | YAML form | (none) |
| `feature_request.md` | Markdown | (none) |
| `config.yml` | YAML config | N/A |

#### Bug Report (`bug_report.yml`)
- **Labels:** `Bug`
- **Fields:**
  - **Operating System** (required dropdown): Linux, MacOS, RaspberryPi OS, Windows 7/10/11, Others
  - **IDE version** (required input): placeholder `e.g Arduino 1.8.15`
  - **Board** (required input): placeholder `e.g Feather nRF52840 Express`
  - **BSP version** (required input): "Release version or github latest"
  - **Sketch** (required textarea): which example or custom code
  - **What happened?** (required textarea)
  - **How to reproduce?** (required textarea): numbered steps format
  - **Debug Log** (optional textarea): "Serial output when IDE's Debug Mode Level to 1 or 2"
  - **Screenshots** (optional textarea)
- **Notable:**
  - Board is a free-text input (not dropdown) -- simpler to maintain than Meshtastic's approach
  - BSP version is an NRF52-specific field
  - Separates "What happened?" from "How to reproduce?" explicitly

#### Feature Request (`feature_request.md`)
- Standard GitHub default template: problem, solution, alternatives, additional context
- **Labels:** `Feature`

#### config.yml
```yaml
blank_issues_enabled: false
contact_links:
  - name: Adafruit Support Forum
    url: https://forums.adafruit.com
    about: If you have other questions or need help, post it here.
  - name: Discussion
    url: https://github.com/adafruit/Adafruit_nRF52_Arduino/discussions
    about: If you have other questions or need help, post it here.
```
- **Two external links:** forum + discussions
- **Blank issues disabled**

### No PR Template

---

## 5. espressif/arduino-esp32 (ESP32 Arduino framework)

**Most comprehensive templates. All YAML forms. Very structured.**

### Issue Templates

| File | Format | Title Prefix |
|------|--------|-------------|
| `Issue-report.yml` | YAML form | (none) |
| `Feature-request.yml` | YAML form | (none) |
| `config.yml` | YAML config | N/A |

#### Issue Report (`Issue-report.yml`)
- **Labels:** `Status: Awaiting triage`
- **Most fields of any project surveyed:**
  - **Board** (required input): placeholder `eg. ESP32 Dev Module, ESP32-S2, LilyGo TTGO LoRa32...`
  - **Device Description** (required textarea): "What development board or other hardware is the chip attached to?"
  - **Hardware Configuration** (required textarea): "Is anything else attached to the development board?"
  - **Version** (required dropdown): ~25 specific version strings from `latest master` down to `v3.0.0` plus `Older versions`
  - **Type** (required dropdown): Task, Bug, Question
  - **IDE Name** (required input)
  - **Operating System** (required input)
  - **Flash frequency** (required input)
  - **PSRAM enabled** (required dropdown): yes/no
  - **Upload speed** (required input)
  - **Description** (required textarea)
  - **Sketch** (required textarea, rendered as `cpp`)
  - **Debug Message** (required textarea, rendered as `plain`)
  - **Other Steps to Reproduce** (optional textarea)
  - **Confirmation checkbox** (required): "I confirm I have checked existing issues, online documentation and Troubleshooting guide."
- **Notable patterns:**
  - Version dropdown with all released versions prevents typos and makes filtering possible
  - Issue type dropdown (Task/Bug/Question) within one template -- rather than separate templates
  - Sketch code uses `render: cpp` for syntax highlighting
  - Debug message uses `render: plain`
  - Hardware-specific fields (flash frequency, PSRAM, upload speed) that seem excessive but help with ESP32 debugging
  - **Mandatory confirmation checkbox** that user checked existing issues/docs

#### Feature Request (`Feature-request.yml`)
- **Labels:** `Type: Feature request`
- **Fields:**
  - **Related area** (required input): `eg. Board support, specific Peripheral, BT, Wifi...`
  - **Hardware specification** (required input): for hardware-dependent features
  - **Is your feature request related to a problem?** (required textarea)
  - **Describe the solution** (required textarea)
  - **Alternatives considered** (optional textarea)
  - **Additional context** (optional textarea)
  - **Confirmation checkbox** (required): checked existing feature requests and contribution guide
- **Notable:** Requires hardware specification even for feature requests

### PR Template (`PULL_REQUEST_TEMPLATE.md`)
- **Checklist section (for deletion after completing):**
  1. [ ] Specific title with component name (eg. "Update of Documentation link on Readme.md")
  2. [ ] Related links (issue that will be closed)
  3. [ ] Update relevant documentation
  4. [ ] Check Contributing guide
  5. [ ] Confirm "Allow edits and access to secrets by maintainers"
- **Body sections:**
  - **Description of Change** -- describe PR and its impact
  - **Test Scenarios** -- hardware and software combinations tested (with example format)
  - **Related links** -- issues, PRs (with `Closes #number` example)
- **Notable:** Explicit test scenario format asking for hardware + software combination

### Other Files
- `CODEOWNERS` -- maps paths to responsible teams (@espressif/arduino-devs, specific maintainers for CI/tools)

#### config.yml
```yaml
blank_issues_enabled: false
contact_links:
  - name: Arduino Core for Espressif Discord Server
    url: https://discord.gg/8xY6e9crwv
    about: Community Discord server for questions and help
```

---

## Summary & Pattern Analysis

### Format Trends

| Project | Issue Format | PR Format |
|---------|-------------|-----------|
| Meshtastic | YAML forms | Markdown |
| RFQuack | Markdown | None |
| RadioLib | Markdown | Markdown |
| Adafruit nRF52 | Mixed (YAML + MD) | None |
| ESP32 Arduino | YAML forms | Markdown |

**Trend:** Newer/larger projects use YAML forms. YAML forms are strongly preferred for bug reports because they enforce structure and make fields required.

### Common Template Types

| Template Type | Meshtastic | RFQuack | RadioLib | Adafruit | ESP32 |
|--------------|------------|---------|----------|----------|-------|
| Bug Report | Y | Y | Y | Y | Y |
| Feature Request | Y | Y | Y | Y | Y |
| New Board/Hardware | Y | - | - | - | - |
| Module Not Working | - | - | Y | - | - |
| Vulnerability | - | Y | - | - | - |
| Regular/Generic Issue | - | - | Y | - | - |

### Commonly Collected Fields (Bug Reports)

| Field | Meshtastic | RFQuack | RadioLib | Adafruit | ESP32 |
|-------|------------|---------|----------|----------|-------|
| Hardware/Board | dropdown | free-text | free-text | free-text input | free-text input |
| Firmware/Library Version | input | free-text | free-text | input | dropdown |
| Description | textarea | section | section | textarea | textarea |
| Steps to Reproduce | (in description) | section | section | textarea | textarea |
| Expected Behavior | (in description) | section | section | - | - |
| Debug/Log Output | textarea (render: Shell) | section | collapsible | textarea | textarea (render: plain) |
| Code/Sketch | - | section | collapsible (render: cpp) | textarea | textarea (render: cpp) |
| OS/IDE | - | free-text | free-text | dropdown + input | input |
| Category/Area | dropdown | - | - | - | - |
| Platform/SOC | (in hardware) | free-text | - | - | - |

### Clever Patterns Worth Adopting

1. **Hardware dropdown with exhaustive board list** (Meshtastic) -- eliminates ambiguity, enables filtering by label; downside is maintenance burden
2. **Version dropdown with all releases** (ESP32 Arduino) -- prevents typos, enables search/filter
3. **`render: Shell` / `render: cpp`** for log and code fields (Meshtastic, ESP32) -- automatic syntax highlighting
4. **Collapsible `<details>` blocks** (RadioLib) -- keeps long debug output from dominating the issue
5. **Mandatory confirmation checkbox** (ESP32 Arduino) -- "I checked existing issues and docs" reduces duplicates
6. **`blank_issues_enabled: false`** (RadioLib, Adafruit, ESP32) -- forces template usage
7. **Contact links in config.yml** -- redirects support questions to Discord/forums/discussions
8. **Separate "module not working" from "software bug"** (RadioLib) -- reduces noise from hardware issues
9. **"New Board" request template** (Meshtastic) -- perfect for multi-hardware firmware projects
10. **PR attestation checklist with specific hardware** (Meshtastic) -- enforces cross-platform testing
11. **Vulnerability template** (RFQuack) -- good security practice for RF/mesh projects
12. **Title prefix convention** (Meshtastic: `[Bug]: `, `[Board]: `, `[Feature Request]: `) -- scannable at a glance
13. **Multi-select dropdowns** for category and hardware (Meshtastic) -- bugs often span multiple areas
14. **Auto-applied labels** (`bug`, `triage`, `enhancement`) -- reduces manual triage work
15. **"Allow edits by maintainers" reminder** (Meshtastic, ESP32) -- important for collaborative PRs

### Anti-Patterns to Avoid

1. **Feature request template that is a copy of bug report** (RFQuack) -- confuses reporters
2. **Too many required fields** (ESP32 Arduino asks for flash frequency, upload speed, PSRAM for every bug) -- may discourage reporting
3. **No PR template at all** (RFQuack, Adafruit) -- leads to inconsistent PR descriptions
4. **Only markdown templates** with no required fields -- reporters skip sections freely
5. **Free-text board/version fields** when a dropdown is feasible -- leads to inconsistent data

### Recommended Template Set for MeshCore

Based on this survey, MeshCore would benefit from:

1. **Bug Report** (YAML form) -- with hardware dropdown (T1000-E, T-Beam variants, Heltec, RAK, etc.), firmware version input, category multi-select (BLE, LoRa, Mesh, CLI, Serial, UI, Other), description, reproduction steps, log output (render: Shell)
2. **Feature Request** (YAML form) -- with platform dropdown (NRF52, ESP32, Cross-Platform), description, use case
3. **New Board Request** (YAML form) -- with SOC dropdown, LoRa IC, product link, description (mirrors Meshtastic)
4. **PR Template** (Markdown) -- with description, test plan, hardware tested checklist, attestation checkbox, "fixes #issue" convention
5. **config.yml** -- disable blank issues, add contact link to Discord/discussions

---

## Raw Template Sources

All templates were fetched via `gh api repos/OWNER/REPO/contents/.github/ISSUE_TEMPLATE/FILENAME` on 2026-02-23. Full content is captured in the analysis above.
