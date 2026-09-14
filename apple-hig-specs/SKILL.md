---
name: apple-hig-specs
description: Supplies the exact numbers from Apple's Human Interface Guidelines — Dynamic Type point sizes and leading for every text style and size category, minimum tap target sizes, layout margins and spacing, control and button sizes, corner radius and concentricity, semantic and system colors, tint and accent color rules, SF Symbols scales, app icon sizes, and accessibility contrast thresholds. Use when a user asks how big a button, font, margin, icon, or touch target should be, what tint or text color to use, or whether a design meets Apple's guidelines.
license: MIT
metadata:
  author: doxuto
  version: "1.0"
  targets: "iOS 26+, iPadOS, macOS, watchOS, tvOS, visionOS"
---

Answer questions about Apple's interface specifications with the actual published numbers, and review designs and code against them.

## Two rules that govern everything else

**1. Don't hardcode what the system provides.** Almost every number below is already available through an API that adapts to the device, the user's text size, the appearance, and the OS version. A hardcoded `17` breaks Dynamic Type; a hardcoded `44` breaks on visionOS; a hardcoded `#007AFF` breaks in dark mode and increased contrast. Use the number to *check* a design, not to write it.

| Instead of | Use |
|---|---|
| `.font(.system(size: 17))` | `.font(.body)` |
| `.padding(16)` at a screen edge | default `.padding()`, or the container's layout margins |
| `.frame(height: 44)` on a button | intrinsic size + `.contentShape` to reach 44 pt |
| `Color(red: 0, green: 0.48, blue: 1)` | `.tint`, `.accentColor`, or a named asset color |
| `Color.black` for text | `.primary` / `Color(.label)` |
| `RoundedRectangle(cornerRadius: 24)` at a screen edge | `ConcentricRectangle()` |
| A fixed nav bar or tab bar height | `safeAreaInsets`, `.safeAreaPadding()` |

**2. In the Liquid Glass era, chrome has no fixed size.** Apple removed most navigation bar, tab bar, and toolbar height specs from the HIG because those elements now resize, minimize on scroll, and adapt to the bottom accessory. Any "nav bar is 44 pt, tab bar is 49 pt" number is stale. Read the safe area instead.

## The numbers people ask for most

**Minimum tap target** — iOS, iPadOS, watchOS: **44×44 pt**. visionOS: **60×60 pt** (and button centers at least 60 pt apart). tvOS: 66×66 pt default focusable size.

**Default body text** — iOS/iPadOS **17 pt**, leading 22 pt, Regular weight. Headline is 17 pt Semibold.

**Minimum text size for custom type styles:**

| Platform | Default | Minimum |
|---|---|---|
| iOS, iPadOS | 17 pt | 11 pt |
| macOS | 13 pt | 10 pt |
| tvOS | 29 pt | 23 pt |
| visionOS | 17 pt | 12 pt |
| watchOS | 16 pt | 12 pt |

**Dynamic Type at the Large (default) size**, iOS and iPadOS:

| Style | Weight | Size | Leading | Emphasized |
|---|---|---|---|---|
| Large Title | Regular | 34 | 41 | Bold |
| Title 1 | Regular | 28 | 34 | Bold |
| Title 2 | Regular | 22 | 28 | Bold |
| Title 3 | Regular | 20 | 25 | Semibold |
| Headline | Semibold | 17 | 22 | Semibold |
| Body | Regular | 17 | 22 | Semibold |
| Callout | Regular | 16 | 21 | Semibold |
| Subhead | Regular | 15 | 20 | Semibold |
| Footnote | Regular | 13 | 18 | Semibold |
| Caption 1 | Regular | 12 | 16 | Semibold |
| Caption 2 | Regular | 11 | 13 | Semibold |

All twelve size categories, including the five accessibility sizes, are in `references/typography.md`. Note the range: Body runs from **14 pt at xSmall to 53 pt at AX5**. A layout built around 17 pt will break — support enlargement to **at least 200%** (140% on watchOS).

**Layout margins**, iPhone, portrait: **16 pt** leading and trailing on the root view, **8 pt** top and bottom. At wider widths the horizontal margin grows to 20 pt. These are the system minimums (`systemMinimumLayoutMargins`), applied as a floor — set a larger custom margin and it is respected, set a smaller one and the system value wins. Read them rather than typing them.

**Spacing between controls** — about **12 pt** of padding around an element that has a bezel, about **24 pt** around the visible edge of one that does not.

**Contrast ratio** (WCAG AA, as Apple states it):

| Text | Minimum |
|---|---|
| Up to 17 pt | 4.5:1 |
| 18 pt and above | 3:1 |
| Bold, any size | 3:1 |

**App icon** — a single **1024×1024 px** square layered source, built in Icon Composer, from which the system generates every size and the six iOS appearances (default, dark, clear light, clear dark, tinted light, tinted dark). Do not hand-draw the rounded rectangle; the system masks it.

## Answering a spec question

1. Give the number, and say which API supplies it so the user does not hardcode it.
2. Name the platform. Nearly every number differs across iOS, macOS, watchOS, tvOS, and visionOS — an unqualified answer is usually wrong for someone.
3. Say what happens at the extremes. A button sized for 17 pt Body is a different button at AX5.
4. If Apple does not publish the number, say so. Several values that circulate widely — fixed nav bar and tab bar heights, a "standard" 12 pt corner radius, an 8 pt grid — are conventions, not published specs. Useful, but label them as conventions.

## Review checklist

When reviewing a design or a screen for HIG compliance:

```
- [ ] Every interactive element reaches 44×44 pt (60×60 pt on visionOS), including icon-only buttons
- [ ] No hardcoded font sizes; text uses semantic styles or scales with UIFontMetrics / @ScaledMetric
- [ ] Layout survives AX5 — rows grow, adjacent views stack, nothing truncates or overlaps
- [ ] No hardcoded colors for text, backgrounds, or separators; semantic colors used for their semantic meaning
- [ ] Text contrast meets 4.5:1 (3:1 at 18 pt or bold), checked in light AND dark AND increased contrast
- [ ] Information is never carried by color alone
- [ ] Content respects safe areas; no fixed chrome heights
- [ ] Corners near the device edge or a container edge are concentric, not a fixed radius
- [ ] Custom fonts declare a minimum size and implement Dynamic Type behavior
- [ ] Light font weights (Ultralight, Thin, Light) avoided for body-size text
- [ ] Reduce Motion, Reduce Transparency, Bold Text, and Increase Contrast are all honored
```

## References

- `references/typography.md` — every Dynamic Type size category with point size and leading, the emphasized weights, platform minimums, and how to scale a custom font correctly.
- `references/layout-and-spacing.md` — layout margins, safe areas, readable content width, size classes, corner radius and concentricity, and the tvOS, visionOS, watchOS and macOS layout numbers.
- `references/controls-and-targets.md` — minimum and default control sizes per platform, button sizes and control size classes, spacing between controls, and why chrome heights are no longer fixed.
- `references/color-and-materials.md` — semantic label, background, fill and separator colors, the system gray RGB values, tint and accent color rules, materials and vibrancy levels, and Liquid Glass color guidance.
- `references/icons-and-symbols.md` — SF Symbols scales, the nine weights, rendering modes, custom interface icon guidance, and app icon requirements.
- `references/accessibility-thresholds.md` — contrast ratios, Dynamic Type enlargement targets, minimum control sizes, control padding, and what each accessibility setting requires of a layout.

## Related skills

This skill supplies the numbers. Three sibling skills cover what to do with them:

- **`swiftui-pro`** — reviewing SwiftUI code, including its platform-releases reference for what shipped in iOS 26 and iOS 27.
- **`swiftui-liquid-glass`** — implementing Liquid Glass correctly.
- **`swiftui-ux-review`** — running a full UI/UX review, including animation and HIG-pattern compliance.
