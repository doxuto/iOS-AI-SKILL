# Icons and symbols

## Contents

- SF Symbols: scales
- SF Symbols: weights
- SF Symbols: rendering modes
- Sizing a symbol
- Custom interface icons
- App icons

## SF Symbols: scales

Three scales, defined relative to the **cap height** of the San Francisco system font:

| Scale | Relationship to text |
|---|---|
| Small | The symbol touches both the cap height and the baseline |
| Medium (default) | The symbol extends slightly above and below the text lines |
| Large | The symbol almost touches both lines at the cap height |

Scale adjusts a symbol's emphasis against adjacent text **without breaking weight matching**. Use it instead of changing the font size when you want a symbol to read larger or smaller than the text it sits next to.

```swift
Label("Favorites", systemImage: "star.fill")
    .imageScale(.large)
```

UIKit: `UIImage.SymbolScale` via `UIImage.SymbolConfiguration`.

## SF Symbols: weights

Nine weights, corresponding one-to-one with the San Francisco font weights: Ultralight, Thin, Light, Regular, Medium, Semibold, Bold, Heavy, Black.

Because they correspond directly, a symbol given the same weight as its adjacent text matches it optically without any manual adjustment. Match the weights of a symbol and its label unless you deliberately want one to dominate.

```swift
Image(systemName: "bolt.fill")
    .font(.body.weight(.semibold))   // matches a semibold label
```

## SF Symbols: rendering modes

| Mode | Behavior |
|---|---|
| **Monochrome** | One color across all layers; interior paths render as transparent cut-outs |
| **Hierarchical** | One color, with opacity varying by each layer's hierarchical level |
| **Palette** | Two or more colors, one per layer; secondary and tertiary can share a color |
| **Multicolor** | The symbol's intrinsic colors, used to reinforce meaning; some layers accept a custom color |

Symbols organize their paths into a **primary**, **secondary** and **tertiary** layer — `cloud.sun.rain.fill`, for example, is cloud (primary), sun (secondary), raindrops (tertiary). Hierarchical and palette modes address those layers.

```swift
Image(systemName: "cloud.sun.rain.fill")
    .symbolRenderingMode(.palette)
    .foregroundStyle(.white, .yellow, .blue)
```

Confirm a rendering mode works in **every** context it appears in — size and background contrast both change how legible a multicolor or hierarchical symbol is. Use system-provided colors so symbols adapt to dark mode, vibrancy, and accessibility settings automatically.

## Sizing a symbol

A symbol placed alongside text should scale with that text, not sit at a fixed point size:

```swift
// Good — scales with Dynamic Type
Image(systemName: "star")
    .font(.body)
    .imageScale(.medium)

// Good — explicit size that still scales
@ScaledMetric(relativeTo: .body) private var iconSize: CGFloat = 20
Image(systemName: "star")
    .resizable().scaledToFit()
    .frame(width: iconSize, height: iconSize)

// Bad — frozen at 20 pt while the label grows to 53 pt
Image(systemName: "star")
    .resizable().scaledToFit()
    .frame(width: 20, height: 20)
```

Remember that a symbol used as a button still needs a 44×44 pt hit area regardless of how small it looks — see `controls-and-targets.md`.

## Custom interface icons

For icons inside the interface, not the app icon:

- **Use a vector format** — PDF or SVG. The system scales it for every display; no separate @2x/@3x assets needed.
- **Keep stroke weight consistent** across the whole icon set, and in general match the weight of adjacent text unless you deliberately want to emphasize one.
- **Adjust dimensions for visual balance.** A lighter-weight icon may need to be slightly taller or wider to look the same size as a heavier one sitting next to it. Optical consistency beats geometric consistency.
- **Pad for optical centering.** An asymmetric icon looks off-center when it is geometrically centered. Add padding inside the asset so that centering the asset geometrically centers the icon optically. The adjustments are small and the effect is large.
- **Don't build a custom selected state** for icons used in standard components — toolbars, tab bars, buttons all update the appearance themselves.

Apple notes no additional platform-specific considerations for interface icons across iOS, iPadOS, tvOS, visionOS and watchOS.

## App icons

**One source, 1024×1024 px, square.** The system masks it to a rounded rectangle — do not draw the rounding yourself, and do not ship a pre-rounded asset.

**Layered.** Build a background layer plus one or more foreground layers. The system applies specular highlights, refraction and translucency to the layers, and those effects adapt with icon size and stay consistent across platforms. A flat single-layer icon gets none of this.

**Six iOS and iPadOS appearances:** default, dark, clear light, clear dark, tinted light, tinted dark.

- Keep the icon's features consistent across all of them — it should read as the same icon.
- The dark variant should be **more subdued** than the default, using complementary colors that reflect the default design. Start from the light icon rather than designing separately.
- Clear and tinted variants are more subdued still.
- Every alternate and variant icon is subject to App Review.

**Tooling.** Use **Icon Composer**, included with Xcode, to define the background layer, place foreground layers, apply effects, preview across system versions and appearances, and export for Xcode. It replaces the old approach of exporting a matrix of PNG sizes.
