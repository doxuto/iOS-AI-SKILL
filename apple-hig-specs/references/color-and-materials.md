# Color and materials

## Contents

- Semantic content colors
- Background colors
- System grays
- Tint and accent color
- Rules for using system colors
- Materials
- Vibrancy
- Liquid Glass color
- Color spaces
- Inclusive color

## Semantic content colors

Use these for foreground content. They adapt to light, dark, and increased-contrast automatically.

| Color | Use for | UIKit | SwiftUI |
|---|---|---|---|
| Label | Primary content text | `UIColor.label` | `.primary` |
| Secondary Label | Secondary content text | `UIColor.secondaryLabel` | `.secondary` |
| Tertiary Label | Tertiary content text | `UIColor.tertiaryLabel` | `Color(.tertiaryLabel)` |
| Quaternary Label | Quaternary content text | `UIColor.quaternaryLabel` | `Color(.quaternaryLabel)` |
| Placeholder Text | Placeholder text in controls | `UIColor.placeholderText` | `Color(.placeholderText)` |
| Separator | Separator that lets underlying content show through | `UIColor.separator` | `Color(.separator)` |
| Opaque Separator | Separator that hides underlying content | `UIColor.opaqueSeparator` | `Color(.opaqueSeparator)` |
| Link | Text that functions as a link | `UIColor.link` | `Color(.link)` |

## Background colors

Two sets, and picking the wrong set is a common source of subtly wrong-looking screens.

**System backgrounds** — for ungrouped content:
`systemBackground`, `secondarySystemBackground`, `tertiarySystemBackground`

**Grouped backgrounds** — for grouped table views and forms:
`systemGroupedBackground`, `secondarySystemGroupedBackground`, `tertiarySystemGroupedBackground`

The levels invert between light and dark mode. In light mode the primary grouped background is grey and the cells are white; in dark mode the primary is black and the cells are grey. That is why hardcoding white cells breaks dark mode — use `secondarySystemGroupedBackground` for the cell and `systemGroupedBackground` for the page.

## System grays

| Name | UIKit | Light (R,G,B) | Dark (R,G,B) |
|---|---|---|---|
| Gray | `systemGray` | 142, 142, 147 | 142, 142, 147 |
| Gray 2 | `systemGray2` | 174, 174, 178 | 99, 99, 102 |
| Gray 3 | `systemGray3` | 199, 199, 204 | 72, 72, 76 |
| Gray 4 | `systemGray4` | 209, 209, 214 | 58, 58, 60 |
| Gray 5 | `systemGray5` | 229, 229, 234 | 44, 44, 46 |
| Gray 6 | `systemGray6` | 242, 242, 247 | 28, 28, 30 |

These values are published so you can design against them, not so you can type them into code. `systemGray` through `systemGray6` run from darkest to lightest in light mode, and the reverse in dark mode.

## Tint and accent color

The tint (accent) color is the single color that marks interactive elements across an app.

```swift
// Whole app
WindowGroup { ContentView().tint(.brandPrimary) }

// A subtree or a single control
Button("Save") { … }.tint(.green)
```

In UIKit, `window.tintColor`, or `tintColor` on a view, which inherits down the hierarchy.

Guidance:

- Define the accent color in an **asset catalog** with light, dark, and high-contrast variants. `AccentColor` in the asset catalog is what Xcode and the system pick up by default.
- One accent color per app. A second "accent" for a different purpose reads as a bug.
- Do not use the accent color for non-interactive content — the whole value of a tint is that tinted means tappable.
- Destructive actions use `role: .destructive` rather than a manually red tint, so the system styles them consistently.

## Rules for using system colors

Two published rules that catch most mistakes:

> **Don't hard-code system color values.** Use APIs like `Color` (SwiftUI) or `UIColor` (UIKit) instead; values fluctuate between releases.

> **Don't redefine semantic meanings of dynamic system colors.** Use `separator` as a separator, not text color; use `secondaryLabel` as text, not background.

Also:

- Make every color work in light, dark, **and increased contrast** — provide all three variants in the asset catalog, not two.
- Avoid using the same color to mean two different things, especially for status or interactivity.
- Test under bright sunlight and in a dim room, and on devices with True Tone and different display profiles.

## Materials

The blur materials, lightest to heaviest:

| Material | Character | Use for |
|---|---|---|
| `ultraThin` | Mostly translucent | Full-screen views needing a light scheme |
| `thin` | More translucent than opaque | Overlays partially obscuring content, light scheme |
| `regular` | Somewhat translucent | Overlays partially obscuring content — the default |
| `thick` | More opaque than translucent | Overlays with a dark scheme |

Thicker materials give better contrast for text and fine detail; thinner ones keep more context from the background visible. Choose by **semantic meaning, not apparent color** — system settings change how each one renders.

## Vibrancy

Vibrant colors are designed to stay legible on top of a material. Use them rather than a plain grey.

**Labels** — `label` (highest contrast, default), `secondaryLabel`, `tertiaryLabel`, `quaternaryLabel`. Avoid `quaternaryLabel` on `thin` or `ultraThin` materials; there is not enough contrast left.

**Fills** — `fill` (default), `secondaryFill`, `tertiaryFill`.

**Separators** — a single default vibrancy value that works on every material.

On visionOS: `label` for standard text, `secondaryLabel` for descriptive text such as footnotes and subtitles, `tertiaryLabel` for inactive elements only.

## Liquid Glass color

Liquid Glass comes in two variants:

**Regular** — blurs and adjusts the luminosity of background content. Use it for anything with significant text: alerts, sidebars, popovers, and any component where the background could hurt legibility.

**Clear** — highly translucent, letting rich background content stay prominent. Use it only for components floating over media such as photos and video. When the underlying content is bright, add a **dark dimming layer at 35% opacity**; over sufficiently dark content, or for AVKit media controls, no dimming is needed.

Published color rules:

- **Apply color sparingly.** Reserve colored Liquid Glass for elements that need emphasis — status indicators, primary actions.
- **Color the background, not the symbol or text.** Apply the accent color to a prominent button's background rather than to its glyph.
- **Don't color the backgrounds of multiple controls.**
- **Avoid similar colors over a colorful background.** Prefer monochromatic toolbars and tab bars there; if you use the accent color, differentiate it clearly.
- **Don't use Liquid Glass in the content layer.** Use standard materials for app backgrounds. The exception is transient interactive elements — sliders, toggles — which take on a glass appearance while active.
- **Use it sparingly overall.** Standard components adopt it automatically; apply it by hand only to custom controls.

## Color spaces

- Apply color profiles to images; sRGB gives accurate results on most displays.
- Use wide color (Display P3) on compatible displays — richer and more saturated, at 16 bits per channel in PNG.
- P3 colors can clip on sRGB displays. Where that matters, ship both versions from an asset catalog.

## Inclusive color

- **Never carry information by color alone.** Add a text label, a glyph, a shape, or a position. This is the most frequently violated guideline in status UI — red dot / green dot with no other cue.
- Check contrast against the thresholds in `accessibility-thresholds.md`; an icon that blends into its background fails for reasons unrelated to color blindness too.
- Colors carry cultural meaning. Red signals danger in some cultures and good fortune in others — Apple's own Stocks app inverts red and green for the Chinese locale. For a multi-market app, treat semantic color as a localizable decision.
