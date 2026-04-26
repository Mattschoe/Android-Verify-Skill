# Interaction Loop

Every verification step follows the same four-phase cycle:

```
1. INSPECT  →  android layout (or screen capture --annotate)
2. DECIDE   →  Find the target element, get its coordinates
3. ACT      →  adb shell input tap/text/swipe/keyevent
4. VERIFY   →  android layout --diff (or screen capture)
```

## Inspecting: `android layout`

Returns a flat JSON array of every UI element currently on screen.

```bash
android layout
```

```json
[
  {"text":"Enable Keyboard","center":"[540,158]","interactions":["clickable","focusable"],"key":3506402},
  {"text":"Activate Keyboard","center":"[540,284]","interactions":["clickable","focusable"],"key":3506402},
  {"text":"hello world","center":"[540,641]","interactions":["clickable","focusable","long-clickable"],"state":["focused"],"key":3506402}
]
```

### Element properties

| Property | Description |
|---|---|
| `text` | Literal text content |
| `resourceId` | Android resource ID |
| `contentDesc` | Accessibility description |
| `interactions` | Supported interactions: `clickable`, `focusable`, `scrollable`, `long-clickable`, `checkable`, `password` |
| `state` | Current state: `checked`, `focused`, `selected` |
| `bounds` | Bounding rectangle `[minX,minY][maxX,maxY]` |
| `center` | Center coordinates `[x,y]` — use these for tap targets |
| `off-screen` | Element exists in hierarchy but isn't visible; may need scrolling |

### `android layout --diff`

Returns only elements that changed since the last `layout` call. Use this after interactions to keep context small.

```bash
android layout --diff
```

If `--diff` returns empty after an action, the UI may still be loading. Wait 2 seconds and retry up to 3 times.

## Inspecting: `android screen capture`

Takes a screenshot of the device screen.

```bash
android screen capture -o /tmp/screen.png
```

With annotation (numbered bounding boxes around every UI element):

```bash
android screen capture --annotate -o /tmp/screen-annotated.png
```

The `--annotate` flag draws numbered pink/magenta bounding boxes around every element. Each element gets a number label that can be used with `screen resolve`.

Always visually examine the PNG before proceeding.

## Resolving: `android screen resolve`

Converts annotated screenshot labels into coordinates.

```bash
android screen resolve --screenshot /tmp/screen-annotated.png --string "tap #6"
# Output: tap 539 158
```

Combine with `adb` for one-shot interaction:

```bash
adb shell input $(android screen resolve --screenshot /tmp/screen-annotated.png --string "tap #6")
```

## Acting: `adb shell input`

```bash
adb shell input tap <x> <y>
adb shell input text "hello%sworld"        # %s = space
adb shell input swipe <x1> <y1> <x2> <y2> <duration_ms>
adb shell input keyevent KEYCODE_BACK
adb shell input keyevent KEYCODE_ENTER
```

## Interaction rules

1. Text input fields **must** have `"focused"` in their `"state"` before entering text. Tap the field first and verify focus.
2. Scrollable elements should be scrolled slowly — use the 5th argument to `swipe` for duration (500ms+).
3. After any action, use `layout --diff` first. If it returns empty, wait 2 seconds, then retry. Up to 3 retries.
4. Use `center` coordinates for tap targets, not `bounds`.

## Example: tap a button and verify navigation

```bash
# 1. Inspect: find the button
android layout
# → {"text":"Enable Keyboard","center":"[540,158]","interactions":["clickable","focusable"]}

# 2. Act: tap it
adb shell input tap 540 158

# 3. Wait briefly for navigation
sleep 1

# 4. Verify: check we're on the right screen
android layout
# → {"content-desc":"On-screen keyboard","center":"[540,136]"} ← Settings page loaded
```

## Example: type text into a field

```bash
# 1. Inspect: find the text field
android layout
# → {"text":"","center":"[540,641]","interactions":["clickable","focusable","long-clickable"]}

# 2. Tap to focus
adb shell input tap 540 641

# 3. Verify focus
android layout --diff
# → {"text":"","center":"[540,641]","state":["focused"]}

# 4. Type text
adb shell input text "hello%sworld"

# 5. Verify
android layout --diff
# → {"text":"hello world","center":"[540,641]","state":["focused"]}
```
