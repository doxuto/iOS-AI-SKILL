# Controls and tap targets

## Contents

- Control sizes per platform
- Reaching 44 pt without a 44 pt box
- Button sizes and control size classes
- Button prominence and roles
- Spacing between controls
- Why chrome heights are no longer fixed
- List rows

## Control sizes per platform

Apple's published default and minimum control sizes:

| Platform | Default | Minimum |
|---|---|---|
| iOS, iPadOS | 44×44 pt | 28×28 pt |
| macOS | 28×28 pt | 20×20 pt |
| tvOS | 66×66 pt | 56×56 pt |
| visionOS | 60×60 pt | 28×28 pt |
| watchOS | 44×44 pt | 28×28 pt |

The **default** is the size to design for. The **minimum** is the floor below which a control is not reliably usable — treat anything between minimum and default as needing a justification, and anything below the minimum as a defect.

On touch platforms the number people quote is 44×44 pt, and that is the right target. It is not the size of the *visual* element — it is the size of the area that responds to touch.

## Reaching 44 pt without a 44 pt box

A 20 pt icon does not need to become a 44 pt icon. It needs a 44 pt hit area.

```swift
// Wrong — pads the visual, making the icon look oversized
Image(systemName: "xmark")
    .padding(12)
    .background(.regularMaterial, in: .circle)

// Right — visual stays small, hit area reaches 44 pt
Button { dismiss() } label: {
    Image(systemName: "xmark")
        .frame(width: 44, height: 44)
        .contentShape(.rect)
}
```

`.contentShape` is the key: without it, the transparent area inside the frame does not accept touches. In UIKit the equivalents are a larger `bounds` with a transparent background, or overriding `point(inside:with:)` / setting `hitTestSlop` style insets.

Common places this is missed:

- Icon-only buttons in a custom toolbar.
- Close buttons in the corner of a card.
- Chevrons and disclosure indicators used as tap targets.
- Text links styled as buttons, where the hit area is only as tall as the line.
- Rows in a custom list where only the label is tappable.
- Segmented control segments with short labels.

Two adjacent 44 pt targets are also too close if their *centers* are 44 pt apart — the targets must not overlap, so their centers need at least 44 pt of separation plus whatever spacing the design calls for.

## Button sizes and control size classes

SwiftUI's `ControlSize` maps to concrete sizes. On visionOS, Apple publishes them:

| Size | visionOS |
|---|---|
| Mini | 28 pt |
| Small | 32 pt |
| Regular | 44 pt |
| Large | 52 pt |
| Extra Large | 64 pt |

```swift
Button("Continue") { … }
    .buttonStyle(.borderedProminent)
    .controlSize(.large)
```

visionOS button shapes by size — circular works at every size; capsule with text only at Small through Large; capsule with text and icon only at Regular and Large; rounded rectangle at Small through Large.

On iOS, use `.controlSize` rather than a `frame`, so the button keeps its intrinsic proportions as text scales. A full-width primary action is the exception:

```swift
Button("Sign in") { … }
    .buttonStyle(.borderedProminent)
    .controlSize(.large)
    .frame(maxWidth: .infinity)
```

## Button prominence and roles

Published guidance:

- **Limit prominent buttons to one or two per view.** More than that and none of them reads as primary.
- **Use style, not size, to distinguish the preferred option** among several buttons.
- **Always include a visible press state** for custom buttons. A custom button with no pressed appearance feels broken.
- On macOS, append a trailing ellipsis when a push button opens another window, view, or app, and include at most one help button per window, in the lower corner opposite the dismissal buttons.
- On watchOS, prefer full-width buttons for primary actions, and give stacked one- and two-line text buttons the same height.

Roles carry meaning and styling for free:

```swift
Button("Delete", role: .destructive) { … }
Button("Cancel", role: .cancel) { … }
```

## Spacing between controls

- About **12 pt** of padding around an element that has a bezel.
- About **24 pt** around the visible edges of an element without a bezel — a bare icon or label needs more breathing room than a bordered button, because its visual edge is its content.
- visionOS: button centers at least **60 pt** apart; 4 pt of padding around buttons 60 pt or larger.
- tvOS: leave room for focus growth — focused elements expand, and tight spacing makes them overlap.

## Why chrome heights are no longer fixed

Numbers like "navigation bar 44 pt, tab bar 49 pt, toolbar 44 pt" appear throughout older material and in model training data. They are no longer specifications, and they are wrong often enough to break layouts:

- Tab bars minimize on scroll (`.tabBarMinimizeBehavior(.onScrollDown)`), changing height mid-gesture.
- A tab view bottom accessory sits above the tab bar at normal size and inline with it when minimized.
- Navigation bars collapse (`.toolbarMinimizeBehavior` on iOS 27) and change height with a large title.
- Liquid Glass chrome floats over content rather than occupying a fixed band.

Apple's current HIG publishes tab bar dimensions only for tvOS (68 pt tall, 46 pt from the top). For every other platform, read the safe area:

```swift
// SwiftUI
.safeAreaInset(edge: .bottom) { Player() }
GeometryReader { proxy in Color.clear.onAppear { print(proxy.safeAreaInsets) } }

// UIKit
view.safeAreaInsets
additionalSafeAreaInsets = UIEdgeInsets(top: 0, left: 0, bottom: 60, right: 0)
```

## List rows

Apple does not publish a fixed row height, and for good reason — a row must grow with the text it holds:

> Table rows may need to grow in height to prevent text cropping or overlap. Single-line rows may need to grow vertically to accommodate multiple lines.

Let rows size themselves. A hardcoded `.frame(height: 44)` on a row is the single most common cause of clipped text at accessibility sizes.

```swift
// Wrong
HStack { … }.frame(height: 44)

// Right
HStack { … }
    .padding(.vertical, 8)
    .frame(minHeight: 44)
```

`minHeight` guarantees the tap target; no maximum lets the row grow. Where a row's content is horizontally adjacent (icon, title, trailing value), consider a `ViewThatFits` so it stacks vertically at large text sizes rather than truncating.
