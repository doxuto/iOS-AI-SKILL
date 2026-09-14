# Layout and spacing

## Contents

- Layout margins
- Safe areas
- Readable content width
- Size classes
- Spacing conventions
- Corner radius and concentricity
- tvOS
- visionOS
- watchOS and macOS

## Layout margins

On iPhone in portrait, a view controller's root view gets **16 pt** leading and trailing margins and **8 pt** top and bottom. At wider widths — iPad, and iPhone in landscape on larger devices — the horizontal margin increases to **20 pt**.

These come from `systemMinimumLayoutMargins`, which behaves as a **floor**:

> If you assign a custom value to `directionalLayoutMargins` of the view controller's root view, the root view's actual margins are set to either your custom values or the minimum values defined by this property, whichever values are greater.

So setting 10 pt where the system minimum is 20 pt gives you 20 pt. To opt out entirely, set `viewRespectsSystemMinimumLayoutMargins = false` — rarely the right call, since it is what keeps content clear of rounded corners and sensor housings.

A plain `UIView` that is not a view controller's root view defaults to 8 pt on all sides.

In SwiftUI, the equivalents are:

```swift
// Default padding — adapts to context and platform
VStack { … }.padding()

// Align with the container's own margins
VStack { … }.safeAreaPadding(.horizontal)

// Read the actual values when you need them
@Environment(\.horizontalSizeClass) private var sizeClass
GeometryReader { proxy in
    let insets = proxy.safeAreaInsets
}
```

Read the margins rather than typing 16. In UIKit, `view.directionalLayoutMargins` or the `layoutMarginsGuide` gives the live value; anchoring to `layoutMarginsGuide` is the whole point of the guide.

## Safe areas

The safe area excludes the status bar, Dynamic Island, home indicator, navigation and tab bars, and on iPad the keyboard and any system chrome. It is not a constant — it changes with device, orientation, presentation, and whether the tab bar is minimized.

Rules:

- Anchor content to the safe area; anchor **backgrounds** to the full bounds with `.ignoresSafeArea()`.
- Never subtract a hardcoded status bar or home indicator height. There is no correct constant.
- A scroll view should extend under the chrome, with `.contentMargins` or `safeAreaInset` supplying the inset, so the scroll edge effect works.

```swift
ScrollView { content }
    .safeAreaInset(edge: .bottom) { FooterBar() }
```

## Readable content width

On a wide screen, a single column of body text that spans the full width is hard to read. UIKit's `readableContentGuide` caps a text column at a comfortable measure — roughly 672 pt at the default text size, narrower at small text sizes and wider at large ones, since it is derived from the current font.

Anchor long-form text to `readableContentGuide` rather than `layoutMarginsGuide`. In SwiftUI, the equivalent is a `frame(maxWidth:)` of roughly 650–700 pt for a reading column, or `.containerRelativeFrame` for proportional layouts. A `Form` and a grouped `List` already do this.

## Size classes

Two dimensions, each **compact** or **regular**. Decide layout from the size class, never from the device model or `UIDevice.current.userInterfaceIdiom`.

> Determine layout based on size classes, not device type or orientation. Consider all possible combinations of size classes. Keep functionality the same as size classes change.

An iPhone in portrait is compact width; most iPads and iPhone Pro Max in landscape are regular width. An iPad app in Split View can be compact width on an iPad — which is exactly why device checks fail.

```swift
@Environment(\.horizontalSizeClass) private var horizontal

var body: some View {
    if horizontal == .compact { VStack { … } } else { HStack { … } }
}
```

## Spacing conventions

Apple does not publish a spacing scale. What it does publish:

- About **12 pt** of padding around an element that has a bezel.
- About **24 pt** around the visible edges of an element without a bezel.
- On visionOS, button centers at least **60 pt** apart, and 4 pt of padding around buttons 60 pt or larger so hover effects do not overlap.

The widely used **8 pt rhythm** (4 / 8 / 16 / 24 / 32) is a convention, not an Apple spec. It is a good convention — it matches the system's own habits and keeps a design consistent — but say so rather than citing it as a guideline.

In SwiftUI, prefer the system's own spacing over a number:

```swift
VStack { … }                    // system default spacing
VStack(spacing: 0) { … }        // explicit when you mean zero
HStack(spacing: nil) { … }      // same as omitting it
```

Scale custom spacing with text so it does not collapse at AX5:

```swift
@ScaledMetric(relativeTo: .body) private var gutter: CGFloat = 16
VStack(spacing: gutter) { … }
```

## Corner radius and concentricity

Two nested rounded rectangles look right when their corners are **concentric** — the inner and outer radii share a center, so the gap between them is even all the way round. A fixed inner radius next to the device's own curve looks wrong, and the device curve differs by model.

iOS 26 gives you this directly with `ConcentricRectangle`, which

> automatically calculates each corner's radius relative to the container shape, so your view adapts correctly across devices and sizes without hard-coded values.

```swift
// Every corner concentric with the container
ConcentricRectangle()
    .fill(.green)
    .padding(8)
    .ignoresSafeArea()

// Mixed: fixed top corners, concentric bottom corners (the Notes format sheet)
ConcentricRectangle(
    uniformTopCorners: .fixed(24),
    uniformBottomCorners: .concentric
)

// Per-corner control, with a floor so a square-cornered device still rounds
ConcentricRectangle(
    topLeadingCorner: .concentric(minimum: 12),
    topTrailingCorner: .fixed(24),
    bottomLeadingCorner: .concentric,
    bottomTrailingCorner: .fixed(0)
)
```

A concentric corner far from the container's corner resolves to a radius of zero — a square corner. Use `.concentric(minimum:)` when you always want a rounded corner.

For a custom container to participate, declare its shape:

```swift
CardContent()
    .containerShape(RoundedRectangle(cornerRadius: 20))
```

Sheets, popovers and system containers already provide a container shape. For the glass shapes themselves, `.glassEffect(in:)` accepts `.rect(cornerRadius:)` with `.containerConcentric`.

Apple publishes no "standard" corner radius. Values like 8, 10 or 12 pt are conventions; match the system component you sit next to rather than picking a number.

## tvOS

Safe area insets: **60 pt** top and bottom, **80 pt** on each side.

Grid layouts — unfocused content width, horizontal spacing, minimum vertical spacing:

| Columns | Width | Horizontal | Vertical |
|---|---|---|---|
| 2 | 860 pt | 40 pt | 100 pt |
| 3 | 560 pt | 40 pt | 100 pt |
| 4 | 410 pt | 40 pt | 100 pt |
| 5 | 320 pt | 40 pt | 100 pt |
| 6 | 260 pt | 40 pt | 100 pt |
| 7 | 217 pt | 40 pt | 100 pt |
| 8 | 184 pt | 40 pt | 100 pt |
| 9 | 160 pt | 40 pt | 100 pt |

Tab bar height is **68 pt**, sitting **46 pt** from the top of the screen. Leave room between focusable elements — they expand when focused, and adjacent items will overlap without padding.

## visionOS

- Minimum control size **60×60 pt**; button centers at least **60 pt** apart.
- 4 pt of padding around buttons of 60 pt or larger, so hover effects do not collide.
- Button size values: Mini 28 pt, Small 32 pt, Regular 44 pt, Large 52 pt, Extra Large 64 pt.
- Support resizing, and set explicit minimum and maximum window sizes.
- Use 3D content sparingly inside a window, and inset it so it does not collide with the window's edges.
- Put supplemental content in an adjacent window rather than an ornament.

## watchOS and macOS

**watchOS** — at most 3 glyph buttons or 2 text buttons side by side in a row. Prefer full-width buttons for primary actions, and give one- and two-line text buttons the same height when they are stacked.

**macOS** — avoid placing controls at the bottom of a window, since people drag windows below the screen edge. Avoid content directly behind the camera housing at the top edge. Image buttons take roughly 10 px of padding between the image and the button's edges.
