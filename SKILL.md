---
name: android-verify
description: >-
  Post-implementation UI verification for Android apps. After implementing a
  feature, verify it works on a real device/emulator using the android CLI —
  no pre-written tests required.
license: Apache-2.0
metadata:
  author: Matthias Nielsen
  keywords:
    - android
    - verify
    - UI verification
    - android-cli
    - journeys
    - layout
    - screenshot
    - post-implementation
---

# Android Verify

After implementing a feature, verify it works on a real device. This skill
standardizes a verification step in the development workflow: inspect the UI,
interact with it, and confirm the feature behaves as expected — all using the
`android` CLI and `adb`. No human writes a test. The output is verified
confidence backed by structured evidence.

## What this skill is NOT

- **Not a test framework.** Maestro, Espresso, and Compose UI tests are for
  humans to author persistent test suites that run in CI. This skill does not
  compete with them.
- **Not a CI gate.** It requires an agent in the loop. It cannot run unattended.
- **Not screenshot regression testing.** It verifies functional behavior (buttons
  work, text appears, navigation happens), not pixel-perfect visual regression.
- **Not exhaustive.** It covers the golden path and key flows. Traditional test
  suites handle exhaustive coverage.

Journey XML files are ephemeral — they structure the agent's verification work,
then they're done. They are not test artifacts for a test suite.

## Prerequisites

Before verification, confirm:

1. **Device connected**: `adb devices` shows a device or emulator
2. **App installed and running**: the app under test is deployed and launched
3. **`android` CLI available**: `android --help` succeeds

## Command reference

Two CLIs are involved: `android` for inspection and screenshots, `adb` for
interaction. **There is no `android tap`** — every tap, swipe, type, and
keyevent goes through `adb shell input`.

### Inspect (`android`)

| Command | Purpose |
|---|---|
| `android layout` | Full JSON of every UI element on screen. Use `center` for tap targets. |
| `android layout --diff` | Only elements that changed since the last `layout` call. Use after every action. |
| `android screen capture -o <path>` | Screenshot to PNG. **Always pass `-o`** — without it the CLI writes to `./screenshot.png` and clobbers prior screenshots. Use a per-step path like `/tmp/verify-<step>.png`. |
| `android screen capture --annotate -o <path>` | Screenshot with numbered bounding boxes around every element. Same `-o` rule applies. |
| `android screen resolve --screenshot <png> --string "tap #<n>"` | **Requires a prior `--annotate` screenshot.** Converts a label like `#6` from that screenshot into `tap x y` coordinates. |

### Act (`adb shell input`)

| Command | Purpose |
|---|---|
| `adb shell input tap <x> <y>` | Tap at coordinates (use `center` from `android layout`). |
| `adb shell input text "hello%sworld"` | Type into the focused field. `%s` = space. |
| `adb shell input swipe <x1> <y1> <x2> <y2> <duration_ms>` | Swipe or scroll. Use 500ms+ for scrolling. |
| `adb shell input keyevent KEYCODE_BACK` | Hardware back button. |
| `adb shell input keyevent KEYCODE_ENTER` | Submit / newline. |
| `adb shell input keyevent KEYCODE_HOME` | Home button. |

### Diagnose (`adb`)

| Command | Purpose |
|---|---|
| `adb devices` | List connected devices/emulators. |
| `adb shell pidof <package>` | Crash check — empty output means the process died. |
| `adb logcat -d --pid=<pid> \| tail -50` | Recent log lines after a crash. |

## Workflow

### Phase 1: Pre-flight

```bash
# Verify device
adb devices

# Verify app is running
adb shell pidof <package.name>

# Take a baseline screenshot
android screen capture -o /tmp/baseline.png
```

### Phase 2: Execute journey

**Source priority:** if the user provides a journey XML file (in the prompt,
attached, or referenced by path), use it verbatim. Only auto-generate a
journey from your implementation context when no journey is supplied. Never
silently substitute your own actions for a user-provided journey. See
[journey format](references/journey-format.md) for the XML spec and
auto-generation guidance.

For each `<action>` in the journey:

1. Inspect current state (`android layout` or `android screen capture`)
2. Perform the action or check the assertion
3. Record result: PASSED, FAILED, or SKIPPED
4. If FAILED, stop execution — remaining actions are SKIPPED

See [interaction loop](references/interaction-loop.md) for the full
inspect → act → verify cycle.

### Phase 3: Report

Output the result as the JSON object below — not a markdown table, not prose.
The structure is part of the skill's contract so downstream tooling and the
developer reviewing the run can parse results consistently.

```json
{
  "journey": "Feature Name Verification",
  "results": [
    {
      "action": "Tap the Submit button",
      "status": "PASSED",
      "commands": ["adb shell input tap 540 800"]
    },
    {
      "action": "Verify success message appears",
      "status": "FAILED",
      "comment": "No success message found. Screen shows: 'Network unavailable'"
    },
    {
      "action": "Check form fields are cleared",
      "status": "SKIPPED"
    }
  ]
}
```

If any action failed, include a screenshot at the point of failure.

## Choosing `layout` vs `screen capture`

Default to `android layout`. Fall back to `android screen capture --annotate`
when layout can't provide what you need.

| Situation | Use |
|---|---|
| Finding elements by text or ID | `layout` |
| Checking state (checked, focused) | `layout` |
| Verifying text content | `layout` |
| System UI (Settings, dialogs) | `layout` |
| Visual appearance (colors, spacing) | `screen capture` |
| Keyboard/IME visibility | `screen capture` |
| `layout` returns empty or errors | `screen capture --annotate` |
| WebView content | `screen capture --annotate` |

**Critical:** IME (keyboard) views are invisible to `layout`. Always use
`screen capture` for keyboard verification. See
[layout vs screenshot](references/layout-vs-screenshot.md) for details.

**`screen resolve` is a follow-up to `--annotate`, not a third inspection
mode.** Once `screen capture --annotate` has labelled elements with numbers
in a PNG, `screen resolve` converts a label like `#6` into `tap x y`
coordinates. If you're already getting coordinates from `android layout`,
you don't need it.

## Auto-generating journeys

After implementing a feature, generate a journey from your knowledge of what
the feature should do:

```xml
<journey name="Delete Account Button">
  <actions>
    <action>Navigate to the Settings screen</action>
    <action>Verify that a "Delete Account" button is visible</action>
    <action>Tap the "Delete Account" button</action>
    <action>Verify that a confirmation dialog appears</action>
    <action>Tap "Cancel" to dismiss the dialog</action>
    <action>Verify that the Settings screen is still shown</action>
  </actions>
</journey>
```

Cover the golden path, include at least one cancel/back path, and end with a
state verification. See [journey format](references/journey-format.md) for the
full spec.

## Interaction rules

1. **Wait for idle with `layout --diff`, not `sleep`** — after every action,
   poll `android layout --diff`. If it returns empty, wait 2 seconds and retry
   up to 3 times. This adapts to fast and slow transitions; a fixed `sleep N &&
   android screen capture` is fragile and should not be the default.
2. **Confirm screen identity before acting on it** — after navigation or app
   launch, run a full `android layout` (not `--diff`) and verify the expected
   screen markers (title, key element, content-desc) are present before
   issuing the next action. Initial layouts can be partially loaded; if
   markers are missing, wait 2 seconds and retry up to 3 times.
3. **Focus before typing** — text fields must have `"focused"` in their `state`
   before using `adb shell input text`.
4. **Scroll slowly** — use 500ms+ duration on `adb shell input swipe`. Fast
   swipes overshoot.
5. **Encode spaces** — use `%s` for spaces in `adb shell input text`.
6. **Tap by coordinate** — read `center` from `android layout` and pass it to
   `adb shell input tap`. There is no semantic `tap --text` or `tap --key`.

See [interaction loop](references/interaction-loop.md) for the full reference.

## Handling failures

- **App crash**: check `adb shell pidof <package>` before and after each action.
  If the process dies, fail immediately and grab logcat.
- **Slow loading**: retry `layout --diff` up to 3 times with 2-second waits.
- **Element not found**: check for `off-screen` elements, try scrolling, then
  fall back to annotated screenshot.
- **IME not rendering**: take a screenshot — Compose-based IME can fail silently
  with no crash or error.

See [common scenarios](references/common-scenarios.md) for detailed handling of
crashes, navigation, scrolling, text input, system dialogs, and emulator startup.

## Guidelines

- The journey XML is the source of truth. If the app disagrees with the journey,
  the app has failed — not the journey.
- Execute each action exactly as written. Do not skip, reorder, or interpret
  intent behind an action.
- Never fix bugs during verification — only report them. Suggestions for fixes
  belong in the journey result summary, not during execution.
- Do not run destructive or cleanup operations (like `gradlew clean`) during
  verification.
- Keep debugging to a minimum. Assume tool output is correct. The goal is to
  determine whether the app passes the journey, not to troubleshoot failures.
- Always take a screenshot on failure — it is the primary evidence for the
  developer.

## Checklist

Before reporting verification results, confirm:

- [ ] Pre-flight passed (device connected, app running, baseline screenshot taken)
- [ ] Every journey action has a status: PASSED, FAILED, or SKIPPED
- [ ] Failed actions include a `comment` explaining what was expected vs observed
- [ ] Failed actions include a screenshot at the point of failure
- [ ] The JSON result summary is complete and well-formed
- [ ] No actions were skipped without a preceding failure
