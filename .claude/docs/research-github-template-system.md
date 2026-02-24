# Research: GitHub Template System — Complete Reference

**Date**: 2026-02-23
**Purpose**: Comprehensive reference for GitHub's template ecosystem (issues, PRs, discussions, community health files), with emphasis on embedded/firmware project best practices applicable to MeshCore.

---

## Table of Contents

1. [Issue Templates](#1-issue-templates)
   - [Legacy Markdown Templates](#11-legacy-markdown-templates)
   - [YAML Form Templates](#12-yaml-form-templates)
   - [Template Chooser (config.yml)](#13-template-chooser-configyml)
2. [Pull Request Templates](#2-pull-request-templates)
3. [Discussion Templates](#3-discussion-templates)
4. [Community Health Files](#4-community-health-files)
5. [Validation Rules and Constraints](#5-validation-rules-and-constraints)
6. [Best Practices for Embedded/Firmware Projects](#6-best-practices-for-embeddedfirmware-projects)
7. [Real-World Examples from Similar Projects](#7-real-world-examples-from-similar-projects)
8. [Sources](#8-sources)

---

## 1. Issue Templates

GitHub supports two formats for issue templates: legacy Markdown (`.md`) and
the newer YAML-based form templates (`.yml`). Both live in
`.github/ISSUE_TEMPLATE/`.

### 1.1 Legacy Markdown Templates

Location: `.github/ISSUE_TEMPLATE/*.md`

These use YAML frontmatter for metadata and Markdown body for the template
content. The issue body is pre-filled but fully editable by the reporter.

```markdown
---
name: Bug Report
about: Create a report to help us improve
title: "[BUG] "
labels: bug, triage
assignees: octocat
---

**Describe the bug**
A clear and concise description of what the bug is.

**To Reproduce**
Steps to reproduce the behavior:
1. Go to '...'
2. Click on '...'

**Expected behavior**
A clear and concise description of what you expected to happen.

**Environment:**
 - Firmware Version: [e.g. 1.2.3]
 - Board: [e.g. T1000-E]
 - OS: [e.g. Android 14]
```

**Frontmatter keys:**

| Key | Type | Required | Description |
|-----|------|----------|-------------|
| `name` | String | Yes | Template name shown in chooser |
| `about` | String | Yes | Description shown in chooser |
| `title` | String | No | Pre-filled issue title |
| `labels` | String/Array | No | Auto-applied labels (must exist in repo) |
| `assignees` | String/Array | No | Auto-assigned users |

**Limitations:**
- No input validation (everything is free-text)
- Users can delete or rearrange sections
- No structured data extraction

### 1.2 YAML Form Templates

Location: `.github/ISSUE_TEMPLATE/*.yml`

These create structured forms with typed fields. Responses are converted to
Markdown in the issue body but the form enforces structure during submission.

#### Top-Level Keys

| Key | Type | Required | Description |
|-----|------|----------|-------------|
| `name` | String | **Yes** | Template name in chooser. Must be unique across all templates. |
| `description` | String | **Yes** | Description shown in chooser interface. |
| `body` | Array | **Yes** | Array of form elements (must have at least 1 non-markdown field). |
| `title` | String | No | Pre-populated issue title (can include placeholders). |
| `labels` | String/Array | No | Auto-applied labels. Non-existent labels silently skipped. |
| `assignees` | String/Array | No | Auto-assigned users. |
| `projects` | String/Array | No | Auto-added to projects. Format: `PROJECT-OWNER/PROJECT-NUMBER`. |
| `type` | String | No | Issue type automatically added (if issue types enabled in repo). |

#### Body Element Types

The `body` array supports five element types:

##### (a) `markdown` — Static display text

Not an input field. Used to provide instructions, section headers, or context.

```yaml
- type: markdown
  attributes:
    value: |
      "## Environment Information"
      Please fill out the fields below so we can reproduce your issue.
```

**Attributes:**

| Attribute | Type | Required | Description |
|-----------|------|----------|-------------|
| `value` | String | **Yes** | Markdown text to render. Wrap headers in quotes to avoid YAML `#` comment parsing. Use `\|` for multiline. |

**Notes:**
- No `id` field (not user input).
- No `validations` block.
- YAML treats `#` as comments — always quote markdown headers.

##### (b) `textarea` — Multi-line text field

```yaml
- type: textarea
  id: description
  attributes:
    label: Bug Description
    description: Describe the bug in detail. Include steps to reproduce.
    placeholder: |
      1. Configure radio with...
      2. Send a message to...
      3. Observe that...
    value: ""
    render: shell
  validations:
    required: true
```

**Attributes:**

| Attribute | Type | Required | Description |
|-----------|------|----------|-------------|
| `label` | String | **Yes** | Brief description shown above the field. |
| `description` | String | No | Guidance text shown below the label. |
| `placeholder` | String | No | Semi-transparent hint text when empty. |
| `value` | String | No | Pre-filled content. |
| `render` | String | No | If set, content is rendered as a code block with this language for syntax highlighting. Uses [GitHub Linguist](https://github.com/github-linguist/linguist) language names (e.g., `shell`, `cpp`, `yaml`, `plain`, `txt`). File attachments are disabled when `render` is set. |

**Validations:**

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `required` | Boolean | `false` | Prevents form submission if field is empty. |

**Notes:**
- Contributors can attach files (drag-and-drop) in textarea fields unless `render` is set.
- When `render` is set, the entire response is wrapped in a fenced code block.

##### (c) `input` — Single-line text field

```yaml
- type: input
  id: firmware-version
  attributes:
    label: Firmware Version
    description: "Run 'version' command in CLI or check device info in the app."
    placeholder: "e.g., 1.5.2-abc1234"
  validations:
    required: true
```

**Attributes:**

| Attribute | Type | Required | Description |
|-----------|------|----------|-------------|
| `label` | String | **Yes** | Brief description shown above the field. |
| `description` | String | No | Guidance/context text. |
| `placeholder` | String | No | Hint text when empty. |
| `value` | String | No | Pre-filled content. |

**Validations:**

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `required` | Boolean | `false` | Prevents submission if empty. |

**Notes:**
- No regex/pattern validation (this has been a community feature request since 2022 but remains unsupported as of February 2026).
- No `maxLength` or `minLength` constraints.

##### (d) `dropdown` — Selection menu

```yaml
- type: dropdown
  id: board
  attributes:
    label: Board / Hardware
    description: Which board are you using?
    multiple: true
    options:
      - T1000-E
      - T-Echo
      - RAK4631
      - Heltec V3
      - Heltec Wireless Paper
      - Station G2
      - ESP32 DIY
      - Other (describe below)
    default: 0
  validations:
    required: true
```

**Attributes:**

| Attribute | Type | Required | Description |
|-----------|------|----------|-------------|
| `label` | String | **Yes** | Menu description. |
| `description` | String | No | Extra context. |
| `multiple` | Boolean | No (default: `false`) | Allow multiple selections. |
| `options` | String Array | **Yes** | List of choices. Must be non-empty; all values must be distinct. |
| `default` | Integer | No | Zero-based index of the pre-selected option. Exclude "None"/"N/A" options when using `default`. |

**Validations:**

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `required` | Boolean | `false` | Enforces at least one selection. |

**Notes:**
- Options cannot include the reserved word "none".
- Options cannot include unquoted YAML boolean values (`true`, `false`, `yes`, `no`, etc.) — these must be quoted.
- All options must be unique within the dropdown.

##### (e) `checkboxes` — Multiple selection checkboxes

```yaml
- type: checkboxes
  id: affected-features
  attributes:
    label: Affected Features
    description: Check all that apply.
    options:
      - label: BLE connectivity
      - label: LoRa mesh routing
      - label: Serial/USB CLI
      - label: WiFi
      - label: Display/UI
      - label: GPS
      - label: Sensors
  validations:
    required: false
```

**Attributes:**

| Attribute | Type | Required | Description |
|-----------|------|----------|-------------|
| `label` | String | **Yes** | Group description shown above checkboxes. |
| `description` | String | No | Markdown-supported guidance text. |
| `options` | Array | **Yes** | Array of checkbox items, each with its own `label` and optional `required`. |

**Each option in `options`:**

| Key | Type | Required | Description |
|-----|------|----------|-------------|
| `label` | String | **Yes** | The checkbox text. Supports limited Markdown (links, bold, italic). |
| `required` | Boolean | No (default: `false`) | Individual checkbox requirement — forces this specific box to be checked. Useful for "I agree to CoC" or "I have searched existing issues" gates. **Only works in public repos.** |

**Validations (group-level):**

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `required` | Boolean | `false` | Requires at least one checkbox in the group to be checked. **Only works in public repos.** |

**Notes:**
- Checkbox labels must be unique among peers and across other input types.
- Individual `required` on checkbox options only works in **public repositories**.

#### Universal Element Keys

For all non-markdown element types:

| Key | Type | Required | Description |
|-----|------|----------|-------------|
| `type` | String | **Yes** | Element type: `markdown`, `textarea`, `input`, `dropdown`, `checkboxes`. |
| `id` | String | Optional (required for all except `markdown`) | Unique identifier. Alphanumeric, hyphens, underscores only. No spaces. Used as the heading in the rendered Markdown output. |
| `attributes` | Object | **Yes** | Key-value pairs defining element properties. |
| `validations` | Object | No | Constraint key-value pairs. |

#### Complete Example: Bug Report Form

```yaml
name: Bug Report
description: Report a bug in MeshCore firmware
title: "[Bug]: "
labels: ["bug", "triage"]
assignees: []
body:
  - type: markdown
    attributes:
      value: |
        Thanks for taking the time to report this bug.
        Please fill out the form below with as much detail as possible.

  - type: checkboxes
    id: prerequisites
    attributes:
      label: Prerequisites
      description: Please confirm the following before submitting.
      options:
        - label: I have searched existing issues and this has not been reported.
          required: true
        - label: I am running the latest firmware version.

  - type: dropdown
    id: board
    attributes:
      label: Board / Hardware
      description: Which board are you using?
      multiple: false
      options:
        - T1000-E
        - T-Echo
        - RAK4631
        - Heltec V3
        - Heltec Wireless Paper
        - Station G2
        - ESP32 (specify below)
        - Other (specify below)
    validations:
      required: true

  - type: dropdown
    id: platform
    attributes:
      label: Platform
      description: Which SoC platform?
      options:
        - NRF52840
        - ESP32
        - ESP32-S3
    validations:
      required: true

  - type: input
    id: firmware-version
    attributes:
      label: Firmware Version
      description: "Firmware version string (e.g., from CLI 'version' command or companion app)."
      placeholder: "e.g., 1.5.2-abc1234"
    validations:
      required: true

  - type: dropdown
    id: category
    attributes:
      label: Category
      description: What area is affected?
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
        - Build System
        - Other
    validations:
      required: true

  - type: textarea
    id: description
    attributes:
      label: Bug Description
      description: Describe the bug clearly. What happened vs. what you expected.
    validations:
      required: true

  - type: textarea
    id: reproduction
    attributes:
      label: Steps to Reproduce
      description: Detailed steps to reproduce the behavior.
      placeholder: |
        1. Configure the device with...
        2. Send a message to...
        3. Observe that...
    validations:
      required: true

  - type: textarea
    id: expected
    attributes:
      label: Expected Behavior
      description: What did you expect to happen?

  - type: textarea
    id: logs
    attributes:
      label: Relevant Log Output
      description: Paste serial/USB log output, BLE debug output, or crash traces.
      render: shell

  - type: input
    id: companion-app
    attributes:
      label: Companion App & Version
      description: "If relevant, which app and version (e.g., Android MeshCore 2.1.0)."

  - type: textarea
    id: additional
    attributes:
      label: Additional Context
      description: Screenshots, radio configuration, mesh topology, or anything else relevant.
```

### 1.3 Template Chooser (config.yml)

Location: `.github/ISSUE_TEMPLATE/config.yml`

Controls the template chooser interface shown when someone clicks "New Issue".

```yaml
blank_issues_enabled: false
contact_links:
  - name: Ask a Question
    url: https://github.com/meshcore-dev/MeshCore/discussions/categories/q-a
    about: Use GitHub Discussions for questions and support.
  - name: Feature Request
    url: https://github.com/meshcore-dev/MeshCore/discussions/categories/ideas
    about: Suggest a feature in Discussions rather than filing an issue.
  - name: Discord Community
    url: https://discord.gg/example
    about: Join the Discord for real-time help.
```

**Keys:**

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `blank_issues_enabled` | Boolean | `true` | When `false`, contributors **must** choose a template or contact link. Cannot open a blank issue. |
| `contact_links` | Array | `[]` | External links displayed in the template chooser alongside templates. |

**Each contact link:**

| Key | Type | Required | Description |
|-----|------|----------|-------------|
| `name` | String | **Yes** | Display name for the link. |
| `url` | String | **Yes** | URL destination (must be a valid URL). |
| `about` | String | **Yes** | Brief description of the link's purpose. |

**Notes:**
- Contact links are rendered alongside issue templates in the chooser UI.
- Setting `blank_issues_enabled: false` is recommended for projects that need structured reports.
- The config applies when merged to the default branch.

---

## 2. Pull Request Templates

PR templates pre-fill the PR description body when a contributor opens a new
pull request.

### Single Template

**Supported locations** (checked in this order):
1. `.github/pull_request_template.md`
2. `pull_request_template.md` (repository root)
3. `docs/pull_request_template.md`

The first template found is used. The filename is case-insensitive on some
platforms, but convention is lowercase or `PULL_REQUEST_TEMPLATE.md`.

### Multiple Templates

Place templates in a subdirectory:

```
.github/PULL_REQUEST_TEMPLATE/
  bug_fix.md
  feature.md
  documentation.md
```

Other valid locations:
- `PULL_REQUEST_TEMPLATE/` (repository root)
- `docs/PULL_REQUEST_TEMPLATE/`

**Usage:** Contributors select a template via the `?template=` URL query
parameter:
```
https://github.com/ORG/REPO/compare/main...feature-branch?template=bug_fix.md
```

**Important notes:**
- There is **no** template chooser UI for PRs (unlike issues). Multiple
  templates require manual URL construction or tooling.
- PR templates do **not** support YAML forms — Markdown only.
- Templates become available after merging to the default branch.

### Example PR Template (Firmware Project)

```markdown
## Summary

<!-- Brief description of what this PR does and why. -->

## Type of Change

- [ ] Bug fix (non-breaking change that fixes an issue)
- [ ] New feature (non-breaking change that adds functionality)
- [ ] Breaking change (fix or feature causing existing functionality to change)
- [ ] Board/variant support
- [ ] Build system / CI
- [ ] Documentation
- [ ] Refactor / code quality

## Related Issues

<!-- Link to related issues: fixes #123, relates to #456 -->

## Hardware Tested

- [ ] NRF52840 (T1000-E, T-Echo, RAK4631, etc.)
- [ ] ESP32
- [ ] ESP32-S3

## Test Plan

<!-- Describe how you tested your changes. -->
- [ ] Compiles without warnings on target environment(s)
- [ ] Tested on physical hardware: _[specify board]_
- [ ] Tested via companion app: _[specify app/platform]_

## Checklist

- [ ] My code follows the project's coding conventions
- [ ] I have tested my changes on relevant hardware
- [ ] I have updated documentation if needed
- [ ] My changes do not introduce new compiler warnings
```

---

## 3. Discussion Templates

Location: `.github/DISCUSSION_TEMPLATE/*.yml`

Discussion category forms use the same form schema as issue templates but with
a reduced set of top-level keys. The filename **must** match the **slug** of the
discussion category (e.g., `announcements.yml` for the "Announcements"
category, `q-a.yml` for the "Q&A" category).

### Top-Level Keys

| Key | Type | Required | Description |
|-----|------|----------|-------------|
| `body` | Array | **Yes** | Form elements (must contain at least 1 non-markdown field). |
| `labels` | String/Array | No | Auto-applied labels. |
| `title` | String | No | Pre-populated discussion title. |

**Differences from issue forms:**
- No `name` or `description` keys (the category itself provides that context).
- No `assignees` or `projects` keys.
- Polls are **not** supported with discussion category forms.

### Body Elements

Identical to issue form body elements: `markdown`, `textarea`, `input`,
`dropdown`, `checkboxes`. Same attributes and validations apply.

### Example: Ideas/Feature Request Discussion Template

File: `.github/DISCUSSION_TEMPLATE/ideas.yml`

```yaml
title: "[Idea]: "
labels: ["enhancement"]
body:
  - type: markdown
    attributes:
      value: |
        Thanks for sharing your idea! Please provide as much detail as possible.

  - type: dropdown
    id: platform
    attributes:
      label: Target Platform
      description: Which platform(s) would this apply to?
      multiple: true
      options:
        - NRF52840
        - ESP32
        - ESP32-S3
        - All platforms
        - Not platform-specific
    validations:
      required: true

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

### Example: Q&A Discussion Template

File: `.github/DISCUSSION_TEMPLATE/q-a.yml`

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
    id: board
    attributes:
      label: Board (if applicable)
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

---

## 4. Community Health Files

GitHub recognizes several "community health files" that appear in the
repository's community profile and sidebar.

### Supported Files

| File | Purpose | Org-level Default? |
|------|---------|-------------------|
| `CODE_OF_CONDUCT.md` | Community engagement standards | Yes |
| `CONTRIBUTING.md` | How to contribute | Yes |
| `SECURITY.md` | Security vulnerability reporting instructions | Yes |
| `SUPPORT.md` | How to get help | Yes |
| `FUNDING.yml` | Sponsor button configuration | Yes |
| `GOVERNANCE.md` | Project governance documentation | Yes |
| Issue/PR templates | Standardized issue/PR creation | Yes |
| `LICENSE` / `LICENSE.md` | Project license | **No** — must be per-repository |

### Search Order (per-repo)

GitHub searches for community health files in this order:
1. `.github/` directory
2. Repository root
3. `docs/` directory

### Organization-Level Defaults

Organizations can create a **public** repository named `.github` containing
default community health files. These apply to **all repositories** in the
organization that do not have their own version.

**Override rule:** If a repository has **any** files in its own
`.github/ISSUE_TEMPLATE/` folder, **none** of the organization-level issue
templates are used (all-or-nothing override).

### FUNDING.yml

Location: `.github/FUNDING.yml`

Adds a "Sponsor" button to the repository. Supports multiple funding platforms.

```yaml
# GitHub Sponsors (up to 4 users + 1 org)
github: [username1, username2]

# External platforms (1 username each)
patreon: username
open_collective: username
ko_fi: username
liberapay: username
issuehunt: username
polar: username
buy_me_a_coffee: username
thanks_dev: u/gh/username

# Tidelift (platform-name/package-name)
tidelift: npm/package-name
# Tidelift platform names: npm, pypi, rubygems, maven, packagist, nuget

# LFX Mentorship (formerly Community Bridge)
community_bridge: project-name

# Custom URLs (up to 4)
custom: ["https://www.paypal.me/example", "https://example.com/donate"]
```

**All supported platform keys:**

| Platform Key | Value Format | Array? |
|-------------|-------------|--------|
| `github` | username(s) | Yes (up to 4 users + 1 org) |
| `patreon` | username | No |
| `open_collective` | username | No |
| `ko_fi` | username | No |
| `liberapay` | username | No |
| `issuehunt` | username | No |
| `polar` | username | No |
| `buy_me_a_coffee` | username | No |
| `thanks_dev` | `u/gh/username` | No |
| `tidelift` | `platform/package` | No |
| `community_bridge` | project-name | No |
| `custom` | URL(s) | Yes (up to 4) |

**Notes:**
- One username per external platform (arrays not supported except for
  `github` and `custom`).
- URLs containing `:` within arrays must be quoted.

### SECURITY.md

Should contain:
- Supported versions table
- How to report a vulnerability (email, not public issues)
- Expected response timeline
- Disclosure policy

### SUPPORT.md

Should direct users to appropriate support channels (Discussions, Discord,
docs) and away from filing issues for questions.

### CONTRIBUTING.md

Should cover:
- Development setup instructions
- Coding standards and conventions
- How to submit changes (PR process)
- Branch conventions
- Testing requirements
- Code review expectations

### GOVERNANCE.md

Should document:
- Decision-making process
- Roles and responsibilities
- How to become a maintainer
- Conflict resolution

---

## 5. Validation Rules and Constraints

### YAML Parsing Rules

- **Forbidden top-level keys:** `y`, `Y`, `yes`, `Yes`, `YES`, `n`, `N`,
  `no`, `No`, `NO`, `true`, `True`, `TRUE`, `false`, `False`, `FALSE`, `on`,
  `On`, `ON`, `off`, `Off`, `OFF` — these are YAML boolean literals and will
  cause parse errors.
- **Boolean option values:** Must be quoted in dropdown options and checkbox
  labels (e.g., `"true"`, `"yes"`).
- **Markdown headers:** Must be quoted because `#` is a YAML comment character.
  Use `"## Section Header"` or the `|` block scalar.

### ID Constraints

- Alphanumeric characters, hyphens (`-`), and underscores (`_`) only.
- No spaces or special characters.
- Must be unique across all body elements in the form.
- The `id` becomes the heading text in the rendered Markdown output.

### Label Constraints

- Labels across input fields must be unique within the form.
- Checkbox labels must be unique among peers and other input types.
- Similar labels that differ only in punctuation may collide.

### Dropdown Option Constraints

- All options must be unique within a dropdown.
- Cannot include the reserved word "none" (case-insensitive).
- Cannot include unquoted YAML boolean values.

### Form Structure Requirements

- The `body` array must contain at least 1 non-markdown field.
- The `name` key cannot be empty or whitespace-only.
- Empty strings are not permitted for required fields.
- Files must have `.yml` extension (not `.yaml`).

### What is NOT Supported (as of February 2026)

- **Regex/pattern validation** on `input` fields — requested since 2022
  ([community discussion #10227](https://github.com/orgs/community/discussions/10227)),
  not implemented.
- **minLength / maxLength** constraints — not available.
- **Conditional logic** (show/hide fields based on other selections) — not
  available.
- **File upload fields** — only supported as drag-and-drop in `textarea`
  (disabled when `render` is set).
- **YAML forms for PRs** — PR templates are Markdown only.
- **Date picker or number input** — not available; use `input` with a
  placeholder to hint at format.

---

## 6. Best Practices for Embedded/Firmware Projects

### Common Issue Categories

Based on analysis of Meshtastic, Marlin, ESPHome, Adafruit nRF52, and
Espressif ESP32 Arduino projects:

| Category | Template Type | Key Fields |
|----------|--------------|------------|
| **Bug Report** | YAML form (`.yml`) | Board, firmware version, platform, reproduction steps, logs |
| **Feature Request** | YAML form or Discussion | Target platform, description, use case |
| **New Board/Hardware Support** | YAML form (`.yml`) | SoC type, LoRa IC, product link, schematic |
| **Build Issue** | YAML form (`.yml`) | OS, toolchain version, PlatformIO version, build target, error log |
| **Module Not Working** | YAML form (`.yml`) | Hardware wiring, debug output, sketch/config |
| **Documentation** | Discussion or simple form | Section affected, what is wrong/missing |

### Critical Information to Collect

#### For Bug Reports:

1. **Board/Hardware** (dropdown) — enumerate all supported boards. Include
   "Other" and "DIY" options. Use `multiple: true` if bugs can affect
   multiple boards.

2. **Platform/SoC** (dropdown) — NRF52840, ESP32, ESP32-S3, etc.

3. **Firmware Version** (input, required) — exact version string including
   commit hash if available.

4. **Category/Area** (dropdown, multiple) — BLE, LoRa, Serial, WiFi, GPS,
   Display, Sensors, Power, Build System.

5. **Reproduction Steps** (textarea, required) — numbered step-by-step.

6. **Log Output** (textarea with `render: shell`) — serial/USB logs, crash
   dumps, BLE debug output.

7. **Companion App** (input) — which app, which version, which mobile OS.

8. **Radio Configuration** (textarea) — frequency, region, power settings,
   mesh topology if relevant.

#### For Build Issues:

1. **Build Target / Environment** (dropdown) — PlatformIO environment name.
2. **PlatformIO Version** (input).
3. **Host OS** (dropdown) — Linux, macOS, Windows, devcontainer.
4. **Toolchain Version** (input) — GCC version, framework version.
5. **Error Output** (textarea with `render: shell`).

#### For Feature Requests:

1. **Target Platform** (dropdown, multiple) — which SoC(s), or cross-platform.
2. **Problem Statement** (textarea) — what problem this solves.
3. **Proposed Solution** (textarea).
4. **Alternatives Considered** (textarea).

#### For New Board Requests:

1. **SoC** (dropdown) — NRF52, ESP32, RP2040, etc.
2. **LoRa IC** (input) — SX1262, SX1276, LR1110, etc.
3. **Product Link** (input, required) — where to find the hardware.
4. **Schematic/Pinout** (textarea) — or link to documentation.

### Template Design Principles

1. **Use `blank_issues_enabled: false`** — forces structured reports, reduces
   low-quality issues.

2. **Use YAML forms over Markdown templates** — structured fields produce
   consistent data that is easier to triage.

3. **Board dropdown should be comprehensive** — enumerate all supported boards
   explicitly. This is the single most important field for firmware projects.

4. **Make firmware version required** — "works on my machine" is useless
   without a version number.

5. **Use `render: shell` for log fields** — keeps logs in code blocks,
   prevents Markdown mangling.

6. **Include prerequisite checkboxes** — "I searched existing issues", "I am
   on the latest firmware". Individual `required: true` on these (public repos
   only).

7. **Redirect questions to Discussions** — use `contact_links` in `config.yml`
   to point support seekers to Discussions or Discord.

8. **Redirect feature requests to Discussions** — keeps the issue tracker
   focused on actionable bugs and board requests.

9. **Title prefixes** (e.g., `[Bug]: `, `[Board]: `) — enable quick visual
   scanning in issue lists.

10. **Auto-apply labels** (`bug`, `triage`, `enhancement`) — reduces manual
    triage work.

---

## 7. Real-World Examples from Similar Projects

### Meshtastic/firmware (LoRa mesh networking firmware — closest comparable project)

**Templates:** `Bug Report.yml`, `New Board.yml`, `feature.yml`

Bug report highlights:
- Category dropdown (multi-select): Hardware Compatibility, BLE, Serial, WiFi, Other
- Hardware dropdown (multi-select): ~40 specific board options including T-Beam, T-Echo, Rak4631, Heltec V3, Seeed Card Tracker T1000-E, DIY, Other
- UI component checkboxes: MUI colorTFT, InkHUD ePaper, OLED slide UI
- Firmware version: input field with placeholder `x.x.x.yyyyyyy`
- Logs: textarea with `render: Shell`
- Labels: `[bug, triage]`
- Title prefix: `[Bug]: `

New Board template:
- SoC dropdown (multi-select): NRF52, ESP32, Other
- LoRa IC: required input
- Product Link: required input
- Description: required textarea
- Labels: `[enhancement, triage]`
- Title prefix: `[Board]: `

Feature Request:
- Platform dropdown (multi-select): NRF52, ESP32, RP2040, Linux Native, Cross-Platform, Other
- Description: required textarea
- Labels: `[enhancement]`
- Title prefix: `[Feature Request]: `

PR template (Markdown):
- Hardware-specific regression attestation checklist (Heltec V3, T-Deck, T-Beam, RAK 4631, Seeed T-1000E, Other)
- Explicit "I don't have this hardware" escape hatch
- Tips section meant to be deleted before submission

### MarlinFirmware/Marlin (3D printer firmware)

Bug report highlights:
- Extensive markdown guidance with links to CoC and contributing guide
- "Did you test latest bugfix branch?" required dropdown (forces acknowledgment)
- Separate fields: Bug Description, Bug Timeline, Expected Behavior, Actual Behavior, Steps to Reproduce
- Hardware inputs: Firmware version, Printer model, Electronics, LCD/Controller, Other add-ons
- Feature-specific dropdowns: Bed Leveling type, Slicer, Host Software
- Required checkbox: "A ZIP file containing your Configuration.h and Configuration_adv.h"
- Labels: `["Bug: Potential ?"]`
- Title prefix: `[BUG] (bug summary)`

config.yml:
- `blank_issues_enabled: false`
- Contact links: Documentation, Facebook group, Discord, Forum, YouTube, Donations

### ESPHome (home automation firmware)

Bug report highlights:
- Version input with description noting format (e.g., "2025.6.0 or 2025.XX.X-dev")
- Installation type dropdown: Home Assistant Add-on, Docker, pip
- Platform dropdown: ESP8266, ESP32, RP2040, BK72XX, RTL87XX, LN882X, Host, Other
- Component name: input field for specific module
- YAML Config: textarea with `render: yaml`
- Logs: textarea with `render: txt`

config.yml:
- `blank_issues_enabled: false`
- Contact links: docs repo, web repo, dashboard repo, feature requests (Discussions), FAQ

PR template (Markdown):
- "What does this implement/fix?" summary section
- Type of change checkboxes: Bugfix, New feature, Breaking change, Developer breaking change, Code quality, Other
- Related issue link and docs PR link
- Test environment checkboxes per platform: ESP32, ESP32 IDF, ESP8266, RP2040, BK72xx, RTL87xx, LN882x, nRF52840
- Example config YAML block
- Checklist: tested locally, tests added, docs updated

### Espressif/arduino-esp32

Bug report highlights:
- Version dropdown with ~25 specific version strings (prevents typos, enables filtering)
- Issue type dropdown within one template: Task, Bug, Question
- Flash frequency, PSRAM enabled, Upload speed fields (ESP32-specific)
- Sketch textarea with `render: cpp`
- Debug message textarea with `render: plain`
- Required confirmation checkbox: "I checked existing issues and docs"

### Adafruit/Adafruit_nRF52_Arduino

Bug report highlights:
- Board as free-text input (simpler to maintain than dropdowns)
- BSP version field (NRF52-specific)
- Debug log with description: "Serial output when IDE's Debug Mode Level to 1 or 2"
- Separates "What happened?" from "How to reproduce?" explicitly

---

## 8. Sources

### GitHub Official Documentation

- [Syntax for issue forms](https://docs.github.com/en/communities/using-templates-to-encourage-useful-issues-and-pull-requests/syntax-for-issue-forms)
- [Syntax for GitHub's form schema](https://docs.github.com/en/communities/using-templates-to-encourage-useful-issues-and-pull-requests/syntax-for-githubs-form-schema)
- [Configuring issue templates for your repository](https://docs.github.com/en/communities/using-templates-to-encourage-useful-issues-and-pull-requests/configuring-issue-templates-for-your-repository)
- [About issue and pull request templates](https://docs.github.com/en/communities/using-templates-to-encourage-useful-issues-and-pull-requests/about-issue-and-pull-request-templates)
- [Common validation errors when creating issue forms](https://docs.github.com/en/communities/using-templates-to-encourage-useful-issues-and-pull-requests/common-validation-errors-when-creating-issue-forms)
- [Creating a pull request template](https://docs.github.com/en/communities/using-templates-to-encourage-useful-issues-and-pull-requests/creating-a-pull-request-template-for-your-repository)
- [Syntax for discussion category forms](https://docs.github.com/en/discussions/managing-discussions-for-your-community/syntax-for-discussion-category-forms)
- [Creating discussion category forms](https://docs.github.com/en/discussions/managing-discussions-for-your-community/creating-discussion-category-forms)
- [Creating a default community health file](https://docs.github.com/en/communities/setting-up-your-project-for-healthy-contributions/creating-a-default-community-health-file)
- [About community profiles](https://docs.github.com/en/communities/setting-up-your-project-for-healthy-contributions/about-community-profiles-for-public-repositories)
- [Displaying a sponsor button (FUNDING.yml)](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/customizing-your-repository/displaying-a-sponsor-button-in-your-repository)

### Firmware Project Templates (fetched 2026-02-23)

- [Meshtastic/firmware .github/ISSUE_TEMPLATE/](https://github.com/meshtastic/firmware/tree/master/.github/ISSUE_TEMPLATE)
- [MarlinFirmware/Marlin .github/ISSUE_TEMPLATE/](https://github.com/MarlinFirmware/Marlin/tree/bugfix-2.1.x/.github/ISSUE_TEMPLATE)
- [ESPHome/esphome .github/ISSUE_TEMPLATE/](https://github.com/esphome/esphome/tree/dev/.github/ISSUE_TEMPLATE)
- [Espressif/arduino-esp32 .github/ISSUE_TEMPLATE/](https://github.com/espressif/arduino-esp32/tree/master/.github/ISSUE_TEMPLATE)
- [Adafruit/Adafruit_nRF52_Arduino .github/ISSUE_TEMPLATE/](https://github.com/adafruit/Adafruit_nRF52_Arduino/tree/master/.github/ISSUE_TEMPLATE)

### Community Discussions & Blog Posts

- [Issue Template forms: Allow validation rules (Discussion #10227)](https://github.com/orgs/community/discussions/10227)
- [Multiple issue and pull request templates (GitHub Blog)](https://github.blog/news-insights/product-news/multiple-issue-and-pull-request-templates/)
- [GitHub Discussions Category Forms (GitHub Blog)](https://github.blog/news-insights/product-news/github-discussions-just-got-better-with-category-forms/)
- [Embedded Artistry: A GitHub Issue Template for Your Projects](https://embeddedartistry.com/blog/2017/08/18/a-github-issue-template-for-your-projects/)
- [Embedded Artistry: A GitHub Pull Request Template for Your Projects](https://embeddedartistry.com/blog/2017/08/04/a-github-pull-request-template-for-your-projects/)
