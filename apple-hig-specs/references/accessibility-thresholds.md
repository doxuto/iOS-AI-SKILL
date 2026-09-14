# Accessibility thresholds

The numeric floors a design has to clear. For the API-level guidance on implementing these, see the `swiftui-pro` skill's accessibility reference; this file is the specifications.

## Contents

- Contrast ratios
- Control sizes
- Text sizes
- Dynamic Type enlargement
- Control padding
- Reduce Motion
- The other settings
- How to check

## Contrast ratios

Apple's stated minimums, matching WCAG Level AA:

| Text size | Weight | Minimum ratio |
|---|---|---|
| Up to 17 pt | Any | **4.5:1** |
| 18 pt and above | Any | **3:1** |
| Any size | Bold | **3:1** |

Note what this means in practice for a default iOS app: Body is 17 pt, so **body text needs 4.5:1** — the looser 3:1 threshold only applies from 18 pt up, or when the text is bold.

Check the ratio in every appearance the text can appear in: light, dark, and **increased contrast** in both. A palette that passes in light mode and fails in dark mode is the usual failure.

Non-text elements that convey information — icons, chart marks, the border of a focused field — need to be distinguishable too. Apple's guidance is to avoid insufficient contrast so icons and text do not blend into their backgrounds; treat 3:1 as the working floor for meaningful non-text elements.

## Control sizes

| Platform | Default | Minimum |
|---|---|---|
| iOS, iPadOS | 44×44 pt | 28×28 pt |
| macOS | 28×28 pt | 20×20 pt |
| tvOS | 66×66 pt | 56×56 pt |
| visionOS | 60×60 pt | 28×28 pt |
| watchOS | 44×44 pt | 28×28 pt |

Design to the **default**. The minimum is a hard floor, not a target.

## Text sizes

For custom type styles:

| Platform | Default | Minimum |
|---|---|---|
| iOS, iPadOS | 17 pt | 11 pt |
| macOS | 13 pt | 10 pt |
| tvOS | 29 pt | 23 pt |
| visionOS | 17 pt | 12 pt |
| watchOS | 16 pt | 12 pt |

Thin weights need to be larger than the minimum to stay legible. In general avoid Ultralight, Thin and Light for anything at body size or smaller.

## Dynamic Type enlargement

> Support text enlargement by **at least 200%** — or **140% in watchOS apps**.

From Body at 17 pt, 200% is 34 pt; the accessibility sizes go further still, to 53 pt at AX5. Test at AX5, not at 200%, since that is what the setting actually offers.

What has to survive:

- **Rows grow.** A single-line row becomes multiple lines. Never fix a row's height.
- **Adjacent views stack.** Horizontally adjacent content needs to reflow vertically. `ViewThatFits` is the cheapest way to express this.
- **Nothing crops or overlaps.** Truncation in a label the user needs to read is a failure, not a graceful degradation.
- **Hit targets stay at 44 pt** even as the label grows — they should grow, not shift.

```swift
#Preview("AX5") {
    ContentView().environment(\.dynamicTypeSize, .accessibility5)
}
```

## Control padding

- About **12 pt** of padding around an element that has a bezel.
- About **24 pt** around the visible edges of an element without a bezel.

A bare glyph or text link needs more clearance than a bordered button, because its visual edge is its content edge — there is no bezel to separate it from its neighbor.

## Reduce Motion

When the setting is on:

- Reduce automatic and repetitive animations.
- Tighten animation springs to reduce bounce.
- Track animations directly to the user's gesture rather than playing them autonomously.
- Avoid animating depth changes on the z-axis.
- Replace x-, y-, and z-axis transitions with **fades**.
- Avoid animating into and out of blurs.

```swift
@Environment(\.accessibilityReduceMotion) private var reduceMotion

.animation(reduceMotion ? nil : .bouncy, value: isExpanded)
.transition(reduceMotion ? .opacity : .move(edge: .bottom))
```

A cross-fade is the standard substitute for a motion transition — it still communicates that something changed, without the movement.

## The other settings

| Setting | Environment key | What the layout must do |
|---|---|---|
| Reduce Transparency | `accessibilityReduceTransparency` | Fall back to opaque backgrounds. iOS 26 adapts Liquid Glass automatically; `.identity` disables it explicitly |
| Increase Contrast | `colorSchemeContrast` | Use the high-contrast variants from the asset catalog; strengthen borders and separators |
| Bold Text | `legibilityWeight` | System fonts adapt automatically; custom fonts must select a heavier face |
| Differentiate Without Color | `accessibilityDifferentiateWithoutColor` | Add shapes, glyphs or text where color alone carried meaning |
| Reduce Motion | `accessibilityReduceMotion` | See above |

Custom fonts get none of these automatically. If an app ships a custom font, it owns Dynamic Type scaling and Bold Text support for it.

## How to check

**Contrast** — Xcode's Accessibility Inspector has a colour contrast calculator; run it against the actual rendered screen in each appearance, not against the hex values in the design file, because materials and vibrancy change the effective ratio.

**Tap targets** — Accessibility Inspector's audit flags elements below the minimum. It will not catch a target that is large enough visually but has no `contentShape`.

**Dynamic Type** — the Accessibility Inspector's settings pane changes the content size category live on a running app, which is faster than rebuilding previews. Sweep from xSmall to AX5 on the busiest screens.

**Everything at once** — Accessibility Inspector's full audit on each major screen catches missing labels, low contrast, small targets, and clipped text in one pass. Run it on a screen before calling the screen done.
