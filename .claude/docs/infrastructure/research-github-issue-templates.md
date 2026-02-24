# Research: GitHub Issue/PR/Discussion Templates for MeshCore

**Date:** 2026-02-23
**Purpose:** Evaluate what GitHub templates MeshCore should adopt, based on the
current repo state, GitHub's template system capabilities, upstream issue/PR
quality patterns, and survey of comparable embedded/mesh firmware projects.

---

## Table of Contents

1. [Current State](#1-current-state)
2. [GitHub Template System Summary](#2-github-template-system-summary)
3. [Upstream Issue & PR Quality Analysis](#3-upstream-issue--pr-quality-analysis)
4. [Survey of Comparable Projects](#4-survey-of-comparable-projects)
5. [Recommended Template Set](#5-recommended-template-set)
6. [Detailed Research Files](#6-detailed-research-files)

---

## 1. Current State

### What exists

- `.github/workflows/` — 3 CI build workflows
- `.github/actions/setup-build-environment/` — custom GHA
- README "Contributing" section — 3 bullet points + "use `dev` as base branch"
- 14 labels defined (bug, enhancement, Firmware, MeshCore App, MeshOS, TDeck /
  Ripple GUI, etc.)
- Discussions enabled (General, Ideas, Polls, Q&A categories)

### What does NOT exist

- Issue templates (no `.github/ISSUE_TEMPLATE/` directory)
- PR template (no `.github/pull_request_template.md`)
- Discussion templates (no `.github/DISCUSSION_TEMPLATE/`)
- `config.yml` for the issue template chooser
- `CONTRIBUTING.md`
- `CODE_OF_CONDUCT.md`
- `SECURITY.md`
- `SUPPORT.md`

### Community health score: **37%** (GitHub metric)

The main gaps are: no code of conduct, no contributing guide, no templates.

---

## 2. GitHub Template System Summary

Full reference: [`research-github-template-system.md`](research-github-template-system.md)

### Issue templates

Two formats, both in `.github/ISSUE_TEMPLATE/`:

| Format | Extension | Validation | Required fields | Structured data |
|--------|-----------|-----------|----------------|----------------|
| Legacy Markdown | `.md` | None — users can delete sections | No | No |
| YAML forms | `.yml` | Yes — enforced at submission | Yes | Yes |

YAML forms support five element types: `markdown` (static text), `textarea`
(multi-line), `input` (single-line), `dropdown` (single/multi-select),
`checkboxes`. Fields can be marked `required: true`. Textarea supports
`render: shell` for syntax-highlighted code blocks.

### Template chooser (`config.yml`)

Controls the "New Issue" page:
- `blank_issues_enabled: false` — forces template selection
- `contact_links` — external links (Discord, Discussions, docs)

### PR templates

Markdown only (no YAML forms). Single file at
`.github/pull_request_template.md` or multiple in
`.github/PULL_REQUEST_TEMPLATE/` (selected via `?template=` URL param — no
chooser UI).

### Discussion templates

`.github/DISCUSSION_TEMPLATE/*.yml` — same YAML form schema as issues. Filename
must match the discussion category slug (e.g., `ideas.yml`, `q-a.yml`).

### Limitations (as of Feb 2026)

- No regex/pattern validation on inputs
- No conditional logic (show/hide fields)
- No YAML forms for PRs
- No minLength/maxLength constraints
- Checkbox `required` only works in public repos

---

## 3. Upstream Issue & PR Quality Analysis

Full analysis: [`research-upstream-repo-state.md`](research-upstream-repo-state.md)

### Issues (525 open)

**Category breakdown** (sample of 20 recent):
- Bug reports: ~50%
- Feature requests: ~25%
- Support/help requests: ~15%
- Mixed/meta: ~10%

**Quality is highly variable.** Without templates to guide structure:

| Quality | Example | What makes it good/bad |
|---------|---------|----------------------|
| Excellent | #1785 (Repeater Lockups) | Hardware details, FW versions, troubleshooting steps, clear hypothesis |
| Good | #1769 (BLE name override) | Repro steps, exact character thresholds, FW versions tested |
| Poor | #1798 (SoCal preset) | Two sentences, no technical justification |
| Minimal | #1790 (loraprs board) | One sentence, no hardware details |

**Commonly missing from bug reports:**
- Steps to reproduce (~70% lack explicit steps)
- Expected vs. actual behavior
- App version / client / OS info
- Logs or serial output
- Region/frequency preset
- Whether it's a regression

**Support requests clog the issue tracker** — many should go to Discussions or
Discord.

### Pull requests

**Bimodal quality distribution:**

| Quality | Example | Structure |
|---------|---------|-----------|
| High | #1803 (Buzzer fix) | Problem, What Changed, Why, Expected Outcome, Compatibility, Testing |
| High | #1778 (Dup suppression) | Summary, Problem, Impact, Scope, Benefits, Drawbacks, ROI |
| Low | #1808 (M5Stack C6L) | One sentence |
| Low | #1762 (BME680 init) | Two sentences + code block, no correctness rationale |

**Commonly missing from PRs:**
- Testing information (most have zero mention)
- Build target verification
- Related issue links
- Impact analysis on other boards/platforms

### Labels

14 defined but almost never applied. Of 40 recent items (20 issues + 20 PRs),
only 1 has labels. Auto-applied labels via templates would fix this.

---

## 4. Survey of Comparable Projects

Full survey: [`research-github-templates.md`](research-github-templates.md)

### Projects surveyed

| Project | Relation to MeshCore | Issue format | PR template |
|---------|---------------------|-------------|-------------|
| **Meshtastic/firmware** | Closest comparable (LoRa mesh FW) | YAML forms (3) | Markdown |
| **espressif/arduino-esp32** | ESP32 framework | YAML forms (2) | Markdown |
| **adafruit/Adafruit_nRF52_Arduino** | NRF52 framework | Mixed (YAML+MD) | None |
| **jgromes/RadioLib** | Multi-platform radio lib | Markdown (4) | Markdown |
| **RFQuack/RFQuack** | RF security tool | Markdown (3) | None |

### Key patterns from mature projects

**1. YAML forms > Markdown templates** — enforces structure, prevents skipped
sections, creates consistent data for triage.

**2. Hardware board dropdown** (Meshtastic: ~40 boards) — eliminates ambiguity,
enables filtering. The single most important field for multi-hardware firmware.

**3. Version dropdown with releases** (ESP32 Arduino: ~25 versions) — prevents
typos, enables search/filter.

**4. `blank_issues_enabled: false`** (RadioLib, Adafruit, ESP32) — forces
template selection, prevents empty issues.

**5. Contact links in `config.yml`** — redirects support questions to
Discord/Discussions/forums. Every mature project does this.

**6. `render: shell` for log fields** — auto syntax highlighting, prevents
Markdown mangling of serial output.

**7. Prerequisite checkboxes** — "I searched existing issues", "I am on the
latest firmware". Reduces duplicates.

**8. Title prefixes** (Meshtastic: `[Bug]: `, `[Board]: `) — scannable at a
glance in issue lists.

**9. Auto-applied labels** — `bug`, `triage`, `enhancement` applied per
template. Eliminates manual label triage.

**10. PR hardware testing checklist** (Meshtastic) — per-platform attestation
checkboxes with explicit "I don't have this hardware" escape hatch.

**11. "New Board" request template** (Meshtastic) — separates board support
requests from generic feature requests. Natural for multi-hardware projects.

**12. Separate "module not working" from "software bug"** (RadioLib) — reduces
noise from hardware/wiring issues.

### Anti-patterns to avoid

- Too many required fields (ESP32 asks flash frequency, upload speed, PSRAM for
  every bug) — discourages reporting
- Feature request template that's a copy of bug report (RFQuack)
- Free-text board/version fields when a dropdown is feasible
- No PR template at all

---

## 5. Recommended Template Set

Based on the research above, MeshCore would benefit from the following templates.
These are ordered by impact.

### 5.1 Issue templates (YAML forms)

#### Bug Report (`bug_report.yml`)

**Purpose:** Structured bug reports with hardware/version/category enforcement.
**Auto-labels:** `bug`, `triage`
**Title prefix:** `[Bug]: `

Fields to collect:
- **Prerequisites** (checkboxes, required) — searched existing issues, running
  latest firmware
- **Board / Hardware** (dropdown, required) — enumerate all supported boards:
  T1000-E, T-Echo, RAK4631, RAK19003, Heltec V3, Heltec Wireless Paper,
  Station G2, T-Deck, T-Beam, T-Beam Supreme, Xiao ESP32-S3, RP2040, DIY,
  Other
- **Platform** (dropdown, required) — NRF52840, ESP32, ESP32-S3, ESP32-C3
- **Firmware Version** (input, required) — with placeholder guidance
- **Firmware Type** (dropdown, required) — Companion Radio, Repeater, Room
  Server
- **Category** (dropdown, multi-select, required) — BLE, LoRa / Mesh Routing,
  Serial / USB, WiFi, Display / UI, GPS, Sensors, Power Management, Build
  System, Other
- **Bug Description** (textarea, required) — what happened vs. expected
- **Steps to Reproduce** (textarea, required)
- **Log Output** (textarea, `render: shell`)
- **Companion App & Version** (input, optional)
- **Additional Context** (textarea, optional) — screenshots, radio config, mesh
  topology

#### New Board Request (`new_board.yml`)

**Purpose:** Hardware support requests separate from feature requests.
**Auto-labels:** `enhancement`, `triage`
**Title prefix:** `[Board]: `

Fields:
- **SoC** (dropdown, required, multi-select) — NRF52840, ESP32, ESP32-S3,
  ESP32-C3, RP2040, Other
- **LoRa IC** (input, required) — SX1262, SX1276, LR1110, etc.
- **Product Link** (input, required)
- **Description** (textarea, required) — board details, pinout, schematic link

#### Build Issue (`build_issue.yml`)

**Purpose:** Compilation/toolchain issues separate from firmware runtime bugs.
**Auto-labels:** `bug`, `triage`
**Title prefix:** `[Build]: `

Fields:
- **Build Target** (input, required) — PlatformIO environment name
- **Host OS** (dropdown, required) — Linux, macOS, Windows, Devcontainer
- **PlatformIO Version** (input, required)
- **Error Output** (textarea, required, `render: shell`)
- **Additional Context** (textarea, optional)

### 5.2 Template chooser (`config.yml`)

```yaml
blank_issues_enabled: false
contact_links:
  - name: Ask a Question
    url: https://github.com/meshcore-dev/MeshCore/discussions/categories/q-a
    about: Use GitHub Discussions for questions and support.
  - name: Feature Request / Idea
    url: https://github.com/meshcore-dev/MeshCore/discussions/categories/ideas
    about: Suggest features and improvements in Discussions.
  - name: Discord Community
    url: <discord-invite-url>
    about: Join the Discord for real-time help and chat.
```

**Rationale for routing feature requests to Discussions:**
- Keeps the issue tracker focused on actionable bugs and board requests
- Feature requests benefit from community discussion before becoming tracked
  work
- Meshtastic and ESPHome both use this pattern
- If a Discussion gains traction, a maintainer can convert it to an issue

### 5.3 PR template (`.github/pull_request_template.md`)

Sections:
- **Summary** — brief description of what and why
- **Type of Change** — checkboxes: Bug fix, New feature, Breaking change,
  Board/variant support, Build system / CI, Refactor
- **Related Issues** — `Fixes #`, `Relates to #`
- **Hardware Tested** — checkboxes per platform: NRF52840 (specify board),
  ESP32 (specify board), ESP32-S3 (specify board), "I do not have hardware to
  test this"
- **Test Plan** — checkboxes: compiles without warnings, tested on physical
  hardware (specify), tested via companion app (specify)
- **Checklist** — follows coding conventions, no new warnings, docs updated if
  needed

### 5.4 Discussion templates

#### Ideas / Feature Requests (`ideas.yml`)

Fields:
- **Target Platform** (dropdown, multi-select) — NRF52840, ESP32, All, Not
  platform-specific
- **Describe your idea** (textarea, required)
- **Alternatives considered** (textarea, optional)
- **Additional context** (textarea, optional)

#### Q&A (`q-a.yml`)

Fields:
- **Topic Area** (dropdown, required) — Getting Started, BLE / Companion App,
  LoRa / Mesh, Build System, Hardware, Other
- **Board** (input, optional)
- **Your Question** (textarea, required)
- **What have you tried?** (textarea, optional)

### 5.5 Community health files (optional, lower priority)

| File | Priority | Notes |
|------|----------|-------|
| `CONTRIBUTING.md` | Medium | Expand README bullets into full guide (setup, PR process, testing, coding style) |
| `SECURITY.md` | Medium | For an RF mesh project, vulnerability reporting matters |
| `SUPPORT.md` | Low | Could redirect to Discussions/Discord |
| `CODE_OF_CONDUCT.md` | Low | Contributor Covenant is standard |

### 5.6 Scope considerations

**Start small, iterate.** A reasonable first PR could include only:
1. Bug Report template (highest impact)
2. New Board template
3. PR template
4. `config.yml` with contact links and `blank_issues_enabled: false`

Discussion templates and community health files can come in follow-up PRs.

---

## 6. Detailed Research Files

- **GitHub template system reference:** [`research-github-template-system.md`](research-github-template-system.md)
  — Complete YAML form schema, all element types and attributes, validation
  rules, limitations, community health file reference, FUNDING.yml format
- **Upstream repo analysis:** [`research-upstream-repo-state.md`](research-upstream-repo-state.md)
  — Detailed issue/PR quality examples, label usage, category breakdown,
  community health score
- **Comparable project survey:** [`research-github-templates.md`](research-github-templates.md)
  — Full template content from Meshtastic, ESP32 Arduino, Adafruit nRF52,
  RadioLib, and RFQuack; pattern analysis table; anti-patterns
