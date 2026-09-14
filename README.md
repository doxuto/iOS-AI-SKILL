# iOS Skills for Claude Code

**12 skills for building iOS and macOS apps with Claude Code** — SwiftUI, Swift concurrency, SwiftData, on-device AI, the Composable Architecture, and a full testing stack.

[github.com/doxuto/iOS-AI-SKILL](https://github.com/doxuto/iOS-AI-SKILL)

Install into `~/.claude/skills/` and Claude loads the matching skill automatically based on what you ask for.

## The skills

### Architecture and state

| Skill | What it does |
|---|---|
| **`tca-pro`** | Composable Architecture 1.26+ — `@Reducer`, `@ObservableState`, effects, dependencies, navigation, `@Shared`, `TestStore`, and every deprecated API with its modern replacement |
| **`swiftdata-pro`** | SwiftData — modeling, predicates, relationships, indexing, migrations, model inheritance |
| **`swift-concurrency-pro`** | Swift 6.2+ concurrency — actors, `Sendable`, structured concurrency, cancellation, strict-concurrency diagnostics |

### UI

| Skill | What it does |
|---|---|
| **`swiftui-pro`** | SwiftUI code review — modern API, deprecation detection, data flow, performance, accessibility, plus what shipped in iOS 26 and iOS 27 |
| **`swiftui-liquid-glass`** | Liquid Glass — all three variants, containers, unions, morphing, `tabViewBottomAccessory`, sheet morphing, common mistakes |
| **`swiftui-ui-patterns`** | 26 component references — NavigationStack, sheets, forms, grids, loading states, theming, deep links |
| **`swiftui-ux-review`** | UI/UX review engine — animations, HIG compliance, accessibility, permission primers |
| **`apple-hig-specs`** | Apple's published numbers — Dynamic Type sizes, tap targets, margins, control sizes, corner concentricity, semantic colors, tint rules, SF Symbols, app icons, contrast thresholds |

### Testing

| Skill | What it does |
|---|---|
| **`ios-test-strategy`** | What to test and at which layer — the pyramid, test doubles, snapshot testing, fixtures, SPM test target layout |
| **`swift-testing-pro`** | Swift Testing — `@Test`, `#expect`, `#require`, parameterized tests, traits, exit tests, attachments, XCTest migration |
| **`xcuitest-pro`** | XCUITest — Robot pattern, launch-state injection, waiting correctly, flakiness diagnosis, test plans and CI |

### AI

| Skill | What it does |
|---|---|
| **`on-device-ai`** | Foundation Models — availability checks, `@Generable`, `@Guide`, the `Tool` protocol, streaming, context window management |

## Install

Install for every project on your machine:

```bash
git clone git@github.com:doxuto/iOS-AI-SKILL.git
mkdir -p ~/.claude/skills
cp -R iOS-AI-SKILL/*/ ~/.claude/skills/
```

Over HTTPS instead, if you have no SSH key set up:

```bash
git clone https://github.com/doxuto/iOS-AI-SKILL.git
```

Install into a single project, so the skills travel with the repo and your team gets them too:

```bash
cd /path/to/your-app
mkdir -p .claude/skills
cp -R /path/to/iOS-AI-SKILL/*/ .claude/skills/
```

Verify with `/skills` in Claude Code — all 12 should be listed. Skills load by their `description`, so you never invoke them by name: ask "review this SwiftUI view" or "how big should this button be" and the matching skill loads itself.

### Updating

```bash
cd /path/to/iOS-AI-SKILL
git pull
cp -R ./*/ ~/.claude/skills/
```

`cp -R` overwrites the skill folders in place and leaves any other skills you have alone.

### Staying on a symlink instead

To keep one working copy and have edits take effect immediately:

```bash
git clone git@github.com:doxuto/iOS-AI-SKILL.git ~/dev/iOS-AI-SKILL
mkdir -p ~/.claude/skills
for skill in ~/dev/iOS-AI-SKILL/*/; do
  ln -sfn "$skill" ~/.claude/skills/"$(basename "$skill")"
done
```

Then `git pull` is the whole update, and editing a `SKILL.md` in the repo changes what Claude loads on the next run.

Each skill is a folder containing `SKILL.md` plus reference files that Claude loads on demand, so an unused reference costs no context.

## How the testing skills fit together

They answer different questions and are meant to be used together:

- **`ios-test-strategy`** — *what* should be tested, and at which layer.
- **`swift-testing-pro`** — *how* to write unit and integration tests.
- **`tca-pro`** (`references/testing.md`) — *how* to test a TCA feature with `TestStore`.
- **`xcuitest-pro`** — *how* to write UI tests.

Swift Testing has no UI-automation or performance-measurement API, so UI tests and `measure(metrics:)` tests stay in XCTest. The two frameworks coexist in one project.

## Where the numbers come from

`apple-hig-specs` carries Apple's published specification tables verbatim — every Dynamic Type size category, per-platform control and text minimums, contrast ratios, system gray values. The other UI skills defer to it rather than restating figures, so there is one place to update when Apple revises them.

Its first rule is that a published number is for *checking* a design, not for writing one: nearly every value has an API that adapts to device, text size, appearance, and OS version. It also flags the specs that no longer exist — fixed navigation bar and tab bar heights were removed from the HIG in the Liquid Glass era, and any code carrying them is already wrong.

## Versions these skills target

| | |
|---|---|
| Platforms | iOS 26 baseline, iOS 27 API gated with `#available` |
| Swift | 6.2+, with Swift 6.3 additions noted where relevant |
| Composable Architecture | 1.26+ |
| Snapshot testing | `swift-snapshot-testing` 1.19+ |

Where a skill documents an API that arrived in a specific release, it says so and shows the availability gate, rather than assuming a deployment target.

## Also worth installing

| Tool | What | Where |
|---|---|---|
| XcodeBuildMCP | Build, test, and archive from Claude | [getsentry/XcodeBuildMCP](https://github.com/getsentry/XcodeBuildMCP) |
| apple-docs-mcp | Search Apple's docs live | [kimsungwhee/apple-docs-mcp](https://github.com/kimsungwhee/apple-docs-mcp) |
| Axiom | Xcode debugging and compiler diagnostics | [CharlesWiltgen/Axiom](https://github.com/CharlesWiltgen/Axiom) |

Apple's own docs are reachable as plain Markdown by appending `.md` to any documentation URL — for example `https://developer.apple.com/documentation/swiftui/glass.md`. That is the fastest way to verify an API actually exists.

## A note on accuracy

Skills that assert API surface are only as good as their verification. The APIs in this bundle were checked against Apple's documentation and, for the Composable Architecture, against the library's own DocC articles and example code.

Things that were found wrong in circulating SwiftUI skill bundles and corrected here:

- `.scrollExtensionMode(.underSidebar)` — does not exist. The real API is `.scrollEdgeEffectStyle(_:for:)`.
- `Glass.prominent` — not a glass variant. `.glassProminent` is a button style.
- `LanguageModelSession.isAvailable` — does not exist. Availability comes from `SystemLanguageModel.default.availability`.
- The Foundation Models `Tool` protocol shape — `Arguments` is a nested `@Generable` type and the method is `call(arguments:)`, not `Input`/`Output: Codable` with `call(with:)`.

If you extend these skills, verify against primary sources before adding an API, and prefer saying "I'm not sure this exists" over writing a plausible-looking name.

## Credits

Bundle assembled and maintained by **[doxuto](https://github.com/doxuto)**.

### Written by doxuto

| Skill | Note |
|---|---|
| `tca-pro` | Written from the Composable Architecture's own DocC articles and example code, following the design principles Point-Free teaches. Point-Free's [The Point-Free Way](https://www.pointfree.co/the-way) skill documents are a separate, subscriber-only product and are not reproduced here. |
| `xcuitest-pro` | — |
| `ios-test-strategy` | — |
| `apple-hig-specs` | Reproduces Apple's published specification tables from the Human Interface Guidelines, cited to their source pages. |

### Upstream skills, included under MIT

| Skills | Original author |
|---|---|
| `swiftui-pro`, `swift-testing-pro`, `swift-concurrency-pro`, `swiftdata-pro` | [Paul Hudson / twostraws](https://github.com/twostraws) |
| `swiftui-liquid-glass`, `swiftui-ux-review`, `swiftui-ui-patterns`, `on-device-ai` | [199 Biotechnologies](https://github.com/199-biotechnologies) |

These eight carry corrections by doxuto — see **A note on accuracy** above — but remain their original authors' work. Each skill's frontmatter records both: `metadata.author` is whoever wrote it, `metadata.maintainer` is who maintains this copy.

## License

MIT.

Copyright © doxuto for `tca-pro`, `xcuitest-pro`, `ios-test-strategy` and `apple-hig-specs`.
Copyright © Paul Hudson and © 199 Biotechnologies for their respective skills, as listed above; their MIT terms carry over to this copy and to any modifications of it.
