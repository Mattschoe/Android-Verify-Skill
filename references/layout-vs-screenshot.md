# Layout vs Screenshot

Default to `android layout` for most operations — it's faster, returns structured data, and doesn't require visual interpretation. Fall back to `android screen capture --annotate` when layout can't provide what you need.

## Decision table

| Situation | Use | Why |
|---|---|---|
| Finding a button, text, or switch | `layout` | Structured search by text/resourceId/contentDesc |
| Checking element state (checked, focused) | `layout` | `state` property is authoritative |
| Verifying text content | `layout` | Exact string match on `text` property |
| Navigating system UI (Settings, dialogs) | `layout` | System views appear in layout dumps |
| Checking visual appearance (colors, spacing) | `screen capture` | Layout has no visual styling data |
| Verifying keyboard/IME is visible | `screen capture` | **IME views are invisible to layout** |
| `layout` returns empty or errors | `screen capture --annotate` | Fallback for WebViews, animations |
| WebView content | `screen capture --annotate` | Web content may not appear in layout |
| Verifying an image or icon | `screen capture` | Layout only has contentDesc, not visual data |

## IME limitation

`android layout` only returns elements from the foreground app's view hierarchy. **IME (keyboard) views rendered by `InputMethodService` do NOT appear.** The keyboard is in a separate window that `layout` does not inspect.

When verifying that a keyboard or IME feature works:
1. Always use `screen capture` — never rely on `layout`
2. If the keyboard area is blank in the screenshot, report this as a failure
3. Grab logcat output to help diagnose rendering issues

## System UI

System Settings pages, system dialogs, and permission prompts DO appear in `layout` output. This includes:
- Toggle switches in Settings (with `checkable`/`checked` state)
- Alert dialog buttons ("OK", "Cancel")
- Permission grant dialogs

You do not need screenshots for system UI interactions.

## Annotated screenshots

`android screen capture --annotate` draws numbered pink/magenta bounding boxes around every detected UI element. Each element gets a number label.

Use annotated screenshots when:
- You need to identify elements that don't appear in `layout`
- You want to visually confirm element positions
- You need coordinates for elements found only in screenshots

To convert a label number to tap coordinates:
```bash
android screen resolve --screenshot /tmp/annotated.png --string "tap #6"
# Output: tap 539 158
```

## When layout fails

`layout` may return empty or fail in these situations:
- **WebViews** — web content doesn't always appear in the view hierarchy
- **Animations in progress** — the tree may be in a transitional state
- **Custom drawing** — `Canvas`-based UI that doesn't use standard views

In these cases, navigate away and back, or wait for animations to complete, then retry. If the issue persists, fall back to `screen capture --annotate`.
