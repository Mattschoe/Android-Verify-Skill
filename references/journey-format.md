# Journey Format

A journey is an XML test script that the agent reads and executes step by step. There is no standalone runner — the agent IS the runner.

## Structure

```xml
<journey name="Feature Name Verification">
  <description>
    What this journey validates
  </description>
  <actions>
    <action>Tap the "Submit" button</action>
    <action>Verify that a success message appears</action>
    <action>Check that the form fields are cleared</action>
  </actions>
</journey>
```

## Action types

### Interaction actions

Specify a UI interaction to perform:

- "Tap the X button"
- "Type 'hello' into the search field"
- "Scroll down in the list"
- "Long-press on the item"

Perform the interaction, then verify the app does not crash or behave unexpectedly. If the interaction cannot be performed as specified (e.g., the element doesn't exist), the journey fails.

If an action contains multiple sub-actions, break them apart and evaluate individually:
```xml
<action>Search for soda and add the first result to the cart</action>
```
Evaluate as two steps: search for soda, then add the first result to the cart.

### Verification actions

Begin with "verify" or "check". Inspect the current state WITHOUT interacting:

- "Verify that the home screen is shown"
- "Check that the counter displays '5'"
- "Verify that the error message is not visible"

Determine the current state by inspecting the screen (using `layout` or `screen capture`). If expectations are not met, the action fails.

A single action may contain multiple expectations:
```xml
<action>Verify that the Settings screen is shown, the toggle is checked, and the title reads "Preferences"</action>
```
This fails if ANY of the three expectations is false.

## Execution protocol

1. Read each `<action>` sequentially
2. For interaction actions: use `layout` to find the element, perform the interaction via `adb`, verify no crash
3. For verification actions: use `layout` (or `screen capture` for visual checks) to inspect state WITHOUT interacting
4. If an action fails, stop execution and mark remaining actions as SKIPPED
5. Produce a JSON summary of results

Execute each step EXACTLY as written, independently of other steps. If an action says "tap the first search result", find the search results and tap the first one — even if you believe you know the intent behind the action.

## Output format

```json
{
  "journey": "Journey Name",
  "results": [
    {
      "action": "Tap the Submit button",
      "status": "PASSED",
      "commands": ["adb shell input tap 540 800"]
    },
    {
      "action": "Verify success message appears",
      "status": "FAILED",
      "comment": "No success message found in layout. Screen shows error: 'Network unavailable'"
    },
    {
      "action": "Check form fields are cleared",
      "status": "SKIPPED"
    }
  ]
}
```

Status values:
- `PASSED` — action executed or expectation met
- `FAILED` — action could not be performed or expectation not met
- `SKIPPED` — not evaluated because a previous action failed

## Handling failure

Failure is acceptable and often expected. Proper reporting of failures is the priority. Keep debugging and troubleshooting to a minimum — assume tools are showing correct output. The goal is to determine if the current app handles the current journey steps. Suggestions for fixes belong in the summary, not during execution.

## Auto-generating journeys

When you just implemented a feature, generate a journey from your implementation context. You know what the feature should do — translate that into actions.

Example: after adding a "Delete Account" button to Settings:

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

Guidelines for auto-generation:
- Cover the golden path (happy case)
- Include at least one "cancel/back" action to verify non-destructive paths
- Verify the starting state before interacting
- End with a verification that the app is in a reasonable state

## Full example: Keyboard activation journey

```xml
<journey name="Markdown Keyboard Setup">
  <description>
    Verify the markdown keyboard can be enabled via system settings,
    then activated via the input method picker, and that text can be
    entered into the test text field.
  </description>
  <actions>
    <action>Tap the "Enable Keyboard" button</action>
    <action>Verify that the on-screen keyboard settings page is shown with "Markdown Keyboard" listed</action>
    <action>Tap the toggle switch next to "Markdown Keyboard" to enable it</action>
    <action>Verify that a confirmation dialog appears about the input method</action>
    <action>Tap "OK" to confirm enabling the keyboard</action>
    <action>Tap "OK" on the follow-up dialog about rebooting</action>
    <action>Verify that the "Markdown Keyboard" toggle is now checked/enabled</action>
    <action>Navigate back to the app</action>
    <action>Verify that the "Enable Keyboard" and "Activate Keyboard" buttons are visible</action>
    <action>Tap the text field and type "hello world"</action>
    <action>Verify that "hello world" appears in the text field</action>
  </actions>
</journey>
```
