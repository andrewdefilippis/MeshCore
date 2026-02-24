# Research: meshcore-dev/MeshCore Upstream Repository State

**Date:** 2026-02-23

## Repository Overview

- **Stars:** 2,053
- **Forks:** 562
- **Open issues (including PRs):** 525
- **Default branch:** `main` (but PRs target `dev`)
- **License:** MIT
- **Discussions:** Enabled (categories: General, Ideas, Polls, Q&A)
- **Community health score:** 37% (GitHub's metric)
- **Has wiki:** Yes
- **Has projects:** Yes

## Community Health Files

Based on the GitHub community profile API:

| File | Present? |
|------|----------|
| README | Yes |
| License (MIT) | Yes |
| Code of Conduct | **No** |
| Contributing guide | **No** (guidelines are in README only) |
| Issue template | **No** |
| PR template | **No** |

The `.github/` directory contains only `actions/` and `workflows/` -- no templates
of any kind.

## Contributing Guidelines (in README)

The README has a "Contributing" section that says:

> Please submit PR's using 'dev' as the base branch! For minor changes just
> submit your PR and I'll try to review it, but for anything more 'impactful'
> please open an Issue first and start a discussion.

It also lists three principles:
1. Keep it simple -- think embedded, keep code concise, no unnecessary layers
2. No dynamic memory allocation except during setup/begin
3. Use the existing brace/indent style (`.clang-format` forthcoming, do NOT
   retroactively reformat)

## Labels

14 labels are defined:

| Label | Description | Color |
|-------|-------------|-------|
| bug | Something isn't working | red |
| documentation | Improvements or additions to documentation | blue |
| duplicate | This issue or pull request already exists | gray |
| enhancement | New feature or request | teal |
| good first issue | Good for newcomers | purple |
| help wanted | Extra attention is needed | green |
| invalid | This doesn't seem right | yellow |
| question | Further information is requested | purple |
| wontfix | This will not be worked on | white |
| feedback | Just general, mixed feedback | light blue |
| TDeck / Ripple GUI | (no description) | blue |
| MeshCore App | (no description) | blue |
| Firmware | (no description) | blue |
| MeshOS | (no description) | blue |

**Label usage is extremely sparse.** Of the 20 most recent issues, only 1
(#1786) has any labels (`question`, `MeshCore App`). The other 19 have zero
labels. Of the 20 most recent PRs, zero have any labels.

The component labels (TDeck/Ripple GUI, MeshCore App, Firmware, MeshOS) exist
but are rarely applied.

## Issues: Patterns and Quality

### Issue Categories Observed

The 20 most recent open issues break down roughly as:

- **Bug reports:** ~10 (e.g., #1805 pin conflict, #1780 reboot corruption,
  #1785 repeater lockups, #1784 low battery flood, #1769 BLE name override,
  #1787 PIN disables touchscreen, #1789 GPS not working, #1788 build error)
- **Feature requests:** ~5 (e.g., #1806 packet error percentage, #1798 region
  preset, #1796 remove location permission, #1783 passive path learning)
- **Help/support requests:** ~3 (e.g., #1804 Vietnam preset change request,
  #1790 board support request, #1800 app navigation issue)
- **Mixed/meta:** ~2 (e.g., #1768 MeshOS improvements, #1775 mesh reliability
  analysis)

### Issue Quality Analysis

**High-quality examples:**

- **#1785 (Repeater Lockups):** Thorough. Includes initial setup, hardware
  details, firmware versions, troubleshooting steps taken, hardware swap
  results, and clear hypothesis about root cause. Formatted with bold headings.
- **#1780 (T1000-E reboot breaks companion):** Concise but has explicit
  reproduction steps. Includes firmware version and recovery method.
- **#1769 (BLE name override):** Good reproduction steps, firmware versions
  tested, exact character count thresholds identified, suggestion for fix.
- **#1805 (ThinkNode M1 LED):** Includes pin conflict analysis with code
  references, multiple duplicated pin definitions, and related GPS observation.

**Low-quality examples:**

- **#1806 (Packet errors as percentage):** One paragraph, no detail on current
  behavior, no mockup of desired behavior.
- **#1798 (SoCal preset):** Two sentences. No technical justification for the
  specific frequency/parameters beyond "other regions have multiple presets."
- **#1804 (Vietnam preset):** Title is the entire issue. Just a frequency
  request with no context.
- **#1790 (loraprs board support):** "Please add support for the loraprs
  board" -- one sentence, no hardware details, no links.
- **#1800 (App navigation):** Describes a UX issue but provides no
  screenshots, device info, or app version.

### Common Information in Issues

**Frequently included:**
- Device/hardware model
- Firmware version
- Basic description of the problem

**Frequently missing:**
- Steps to reproduce (only ~30% of bug reports have explicit steps)
- Expected vs. actual behavior (rarely structured this way)
- App version / client information
- Operating system (for app issues)
- Logs or serial output
- Screenshots
- Region/frequency preset in use
- Whether the issue is a regression (which version it last worked on)

### Issue Formatting

- No consistent structure across issues
- Some use markdown formatting (bold, code blocks), many are plain text
- No one uses the labels -- the maintainer occasionally applies them
- Bug reports, feature requests, and support questions all use the same
  undifferentiated issue form
- Feature requests frequently lack: use case justification, acceptance
  criteria, consideration of alternatives
- Support requests that should be discussions end up as issues

## Pull Requests: Patterns and Quality

### PR Quality Spectrum

**High-quality PRs (detailed, structured):**

- **#1803 (Buzzer startup fix):** Has Problem, What Changed, Why, Expected
  Outcome (with table), Compatibility analysis, Testing section. References
  related issues. Clear before/after comparison.
- **#1801 (BLE name fix):** Has Problem, What Changed (with examples table),
  Why, Expected Outcome, Testing, References section with links to SoftDevice
  headers and Bluefruit source. Root cause analysis included.
- **#1782 (Passive path candidates):** Has Summary, Theory/Motivation,
  Implementation details (4 subsections), Benefits, Drawbacks/Failure Modes,
  Asymmetric Link Risk analysis. Very thorough technical explanation.
- **#1778 (Duplicate suppression):** Has Summary, Problem, Impact, Scope,
  Description, How It Addresses the Problem, Scope of Fix, Benefits,
  Drawbacks/Tradeoffs, New Complexity, ROI. Extremely structured.

**Low-quality PRs (minimal description):**

- **#1808 (M5Stack C6L build flags):** One sentence: "Enabled USB-CDC on boot
  for M5Stack_Unit_C6L_companion_radio_usb to fix serial connection issues."
  No context on what was broken or how to test.
- **#1762 (BME680 init):** Two sentences plus a code block. No explanation of
  why the change is correct or what happens without it.
- **#1749 (Station G1 support):** "it boots, and is configurable over
  usb/radio... it's screaming about i2c issues... more to come." Clearly WIP
  but no context.
- **#1760 (RAK3401 build flags):** No description body at all based on its
  sparse listing.

### Common Information in PRs

**High-quality PRs include:**
- Problem statement / root cause
- What changed and why
- Expected outcome / before-after comparison
- Testing performed (build targets, hardware testing)
- Related issues linked
- Compatibility/impact analysis

**Frequently missing (especially in smaller PRs):**
- Testing information (most PRs have zero mention of testing)
- Build target verification
- Related issue links
- Impact analysis on other boards/platforms
- How to review or manually test the change

### PR Labels

Zero labels on any of the 20 most recent PRs. Labels are not used for PRs at all.

## Discussions

Discussions are enabled with at least these categories: General, Ideas, Polls, Q&A.

Recent discussion examples:
- #1679 "Question: Why is noise_floor clamped to -120 dB?" (General)
- #1613 "The Case for Expanding Repeater IDs Beyond 1 Byte" (Ideas)
- #1614 "V2 Prefixes" (Polls)
- #1618 "Help with Telemetry on QWIIC Module" (Q&A)
- #1621 "V2 - reduce self repeater traffic" (Ideas)

This suggests some support questions and brainstorming go to discussions, but
many support questions still end up as issues.

## Template Search Results

- Searching for "template" in issues returned 5 results, none of which were
  about issue/PR templates themselves (they were about code style, USB serial,
  ESPHome integration, encryption, and NFC).
- Searching for "issue template" returned zero results.
- No `.github/ISSUE_TEMPLATE/` directory exists.
- No `.github/PULL_REQUEST_TEMPLATE.md` exists.

## Key Findings Summary

1. **No templates exist at all.** The repository has zero issue templates and
   zero PR templates. All issues and PRs use a blank form.

2. **Label usage is nearly zero.** Despite having 14 labels defined (including
   useful component labels like Firmware, MeshCore App, MeshOS, TDeck/Ripple
   GUI), they are almost never applied. Of the 40 most recent items
   (20 issues + 20 PRs), only 1 issue has labels.

3. **Issue quality is highly variable.** Ranges from excellent multi-section
   bug reports with reproduction steps (#1785) to one-sentence feature requests
   with no context (#1790). There is no structure to guide contributors.

4. **PR quality is bimodal.** A subset of contributors (notably `robekl` and
   `andrewdefilippis`) produce highly structured PRs with problem/change/why/
   testing sections. The majority of PRs have minimal descriptions -- sometimes
   just one sentence or a title only.

5. **Bug reports vs. feature requests vs. support questions are
   undifferentiated.** All use the same blank issue form. Many "issues" are
   actually support requests or questions that would be better as discussions.

6. **Contributing guide is README-only.** The three bullet points in the README
   are the entire contributor guidance. No CONTRIBUTING.md, no code of conduct.

7. **Testing is rarely documented.** Most PRs do not mention what was tested,
   what build targets were verified, or what hardware was used.

8. **The maintainer's README asks for "impactful" changes to start with an
   issue discussion first**, but there is no mechanism to enforce or guide this.

9. **The repo has a 37% community health score**, which is low. The main gaps
   are: no code of conduct, no contributing guide, no issue templates, no PR
   templates.

10. **Discussions are underutilized.** Support questions and general feedback
    frequently land as issues instead of being directed to the Discussions or
    Discord channels.
