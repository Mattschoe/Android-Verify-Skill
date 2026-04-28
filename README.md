## android-verify

An [agent skill](https://agentskills.io/home) for post-implementation UI verification of Android
apps. After an AI agent implements a feature, this skill teaches it to verify the feature works on a
real device or emulator — no pre-written tests required.

> **Quick Start:**
> ```bash
> npx skills add git@github.com:Mattschoe/Android-Verify-Skill.git
> ```

### How it works

1. **Human:** "Add a settings screen with a dark mode toggle"
2. **Agent:** Implements the feature in Kotlin/Compose
3. **Agent:** Deploys to a device, opens the app, navigates to Settings, taps the toggle, verifies
   the theme changes — all using the `android` CLI
4. **Agent:** "Done. Feature works. Here's what I verified."

No human wrote a test. No Maestro YAML. No Espresso rule. The agent that wrote the code is the same
agent verifying it, using the context it already had from building the feature.

### Where it fits

| Layer | Tool | Runs in CI | Needs human-authored tests |
|---|---|---|---|
| Unit tests | JUnit | Yes | Yes |
| Component tests | Compose UI Test | Yes | Yes |
| Integration tests | Espresso | Yes | Yes |
| E2E flow tests | Maestro | Yes | Yes |
| Screenshot tests | Paparazzi | Yes | Yes |
| **Dev-time verification** | **android-verify** | **No** | **No** |

Every other tool requires someone to write the test first. This skill generates verification from
implementation context.

### Prerequisites

- [Android CLI](https://developer.android.com/tools/agents/android-cli) installed
- A connected device or running emulator (`adb devices`)
- An Android app deployed to the device

### Install

```bash
npx skills add git@github.com:Mattschoe/Android-Verify-Skill.git
```

Or install globally (user-level, available in all projects):

```bash
npx skills add git@github.com:Mattschoe/Android-Verify-Skill.git -g
```

#### Manual install (Claude Code)

Clone and copy to your Claude Code skills directory:

```bash
git clone git@github.com:Mattschoe/Android-Verify-Skill.git
cp -r Android-Verify-Skill ~/.claude/skills/android-verify
```

#### Other agents

Copy `SKILL.md` and the `references/` directory to your agent's skills directory.

### Recommended: Add a CLAUDE.md instruction

Claude Code won't always invoke this skill automatically after implementing a feature or fix. To
ensure consistent verification, add the following to your project's `CLAUDE.md`:

```
After completing any Android UI implementation (features, fixes, or changes that affect what the
user sees or interacts with), run /android-verify before reporting the task as done — as long as
the change can be verified through the android CLI on a connected device or emulator.
```

This makes verification a natural part of the agent's workflow rather than something you have to
remember to ask for.

### Documentation

- [SKILL.md](SKILL.md) — Full skill specification
- [references/interaction-loop.md](references/interaction-loop.md) — The inspect → act → verify
  cycle
- [references/journey-format.md](references/journey-format.md) — Journey XML format and execution
  protocol
- [references/layout-vs-screenshot.md](references/layout-vs-screenshot.md) — When to use `layout`
  vs `screen capture`
- [references/common-scenarios.md](references/common-scenarios.md) — Handling crashes, timing,
  scrolling, IME, and more

### Contributing

Issues and pull requests are welcome.

### License

Licensed under the [Apache License 2.0](LICENSE.txt).
