# Common Scenarios

## App crashes

Check for crashes before and after each action:

```bash
adb shell pidof <package.name>
```

If the process ID disappears after an action, the app crashed. The journey fails immediately. Grab the crash reason:

```bash
adb logcat -d --pid=<last_known_pid> | tail -50
```

Include the crash stacktrace in the journey result comment.

## Slow loading

After an action, if `android layout --diff` returns empty:

1. Wait 2 seconds
2. Run `android layout --diff` again
3. Repeat up to 3 times
4. If still empty after 3 retries, either the action had no visible effect (possible failure) or the UI change is too subtle for diff to catch — take a full `android layout` or screenshot to investigate

Content-heavy screens (lists, images, network data) may take longer. Adjust wait times based on what the screen is loading.

## Navigation

### Going back
```bash
adb shell input keyevent KEYCODE_BACK
```

### Jumping to a specific screen
```bash
adb shell am start -n <package>/<fully.qualified.ActivityName>
```

### Going home
```bash
adb shell input keyevent KEYCODE_HOME
```

### Opening recent apps
```bash
adb shell input keyevent KEYCODE_APP_SWITCH
```

After navigation, always run `android layout` to confirm you arrived at the expected screen.

## Scrolling

If an element isn't found in `layout`, check for:
1. An element with `"off-screen": true` — it exists but needs scrolling
2. A nearby element with `"scrollable"` in its `interactions`

To scroll, use `adb shell input swipe` with a slow duration:

```bash
# Scroll down (swipe up)
adb shell input swipe 540 1500 540 500 500

# Scroll up (swipe down)
adb shell input swipe 540 500 540 1500 500
```

The 5th argument is duration in milliseconds — use 500ms or more. Fast swipes can overshoot.

After scrolling, use `android layout --diff` to check if the target element appeared.

## Text input

1. **Tap the field** to give it focus:
   ```bash
   adb shell input tap <x> <y>
   ```
2. **Verify focus** — the element must have `"focused"` in its `state`:
   ```bash
   android layout --diff
   # Look for: "state":["focused"]
   ```
3. **Type text**:
   ```bash
   adb shell input text "hello%sworld"
   ```

Special characters:
- Spaces → `%s`
- Other special characters may need shell escaping

`adb shell input text` bypasses the IME — it doesn't test the keyboard itself, just text entry into the field. To test actual keyboard key presses, tap key coordinates from a screenshot.

## IME / keyboard verification

`android layout` does **not** show keyboard views. The `InputMethodService` renders in a separate window.

To verify a keyboard is visible:
1. Take a screenshot: `android screen capture -o /tmp/ime-check.png`
2. Visually examine the bottom portion of the screen
3. If the keyboard area is blank, the IME failed to render — report as failure

If verifying specific keyboard keys or layout:
1. Use `android screen capture --annotate -o /tmp/ime-annotated.png`
2. Visually identify key positions from the annotated screenshot
3. Use `android screen resolve` to get coordinates for tapping specific keys

Compose-based IME rendering can fail silently — the service may be bound (`mInputShown=true`) but the window not visible (`mWindowVisible=false`). No crash, no error. Screenshots are the only way to detect this.

## System dialogs

System dialogs (permission prompts, security warnings, confirmation dialogs) DO appear in `android layout`. Handle them like any other UI element:

```bash
# Inspect the dialog
android layout
# → {"text":"OK","center":"[540,900]","interactions":["clickable","focusable"]}

# Tap the button
adb shell input tap 540 900
```

Some system interactions trigger multiple dialogs in sequence. After dismissing one, run `android layout` again to check for the next.

## Emulator startup

`android emulator start` may time out on first boot. Instead of relying on the CLI timeout, poll for readiness:

```bash
# Check if any device is connected
adb devices

# Check if boot completed
adb shell getprop sys.boot_completed
# Returns "1" when ready
```

## Build toolchain

Some Android projects require specific JDK versions. If the build fails with JDK errors, try setting JAVA_HOME:

```bash
JAVA_HOME=/usr/lib/jvm/java-21-openjdk ./gradlew :app:assembleDebug
```

AGP 8.x typically requires JDK 17 or 21. AGP 9.x may have different requirements. Check the project's `gradle.properties` or build files for compatibility notes.
