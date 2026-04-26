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

Either receive a journey XML file, or auto-generate one from the feature you
just implemented. See [journey format](references/journey-format.md) for the
XML spec and auto-generation guidance.

For each `<action>` in the journey:

1. Inspect current state (`android layout` or `android screen capture`)
2. Perform the action or check the assertion
3. Record result: PASSED, FAILED, or SKIPPED
4. If FAILED, stop execution — remaining actions are SKIPPED

See [interaction loop](references/interaction-loop.md) for the full
inspect → act → verify cycle.

### Phase 3: Report

Output a JSON summary:

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

1. **Focus before typing** — text fields must have `"focused"` in their `state`
   before using `adb shell input text`
2. **Scroll slowly** — use 500ms+ duration on `adb shell input swipe`
3. **Use `layout --diff`** — after actions, check only what changed to keep
   context small
4. **Retry on empty diff** — if `layout --diff` returns nothing after an action,
   wait 2 seconds and retry up to 3 times
5. **Encode spaces** — use `%s` for spaces in `adb shell input text`

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
