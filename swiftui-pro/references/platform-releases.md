# What shipped in which release

Every API below must be gated with `#available` when the deployment target is lower, and given a fallback. This file exists because model training data lags OS releases: these are the APIs most often either missed entirely or invented.

## Contents

- iOS 26 additions
- iOS 27 additions
- Availability gating
- Commonly hallucinated API

## iOS 26 additions

**Liquid Glass**

| API | Notes |
|---|---|
| `.glassEffect(_:in:isEnabled:)` | Shape defaults to a capsule |
| `Glass.regular`, `.clear`, `.identity` | The only three variants |
| `.tint(_:)`, `.interactive()` on `Glass` | `interactive()` is iOS-only |
| `GlassEffectContainer(spacing:)` | Required when multiple glass views coexist |
| `.glassEffectID(_:in:)` | Morphing between glass shapes, with `@Namespace` |
| `.glassEffectUnion(id:namespace:)` | Merges same-shape, same-variant effects into one shape |
| `.buttonStyle(.glass)` / `.buttonStyle(.glassProminent)` | Button styles, not `Glass` variants |

**Layout and chrome**

| API | Notes |
|---|---|
| `.tabViewBottomAccessory { }` | Persistent control above the tab bar; placement varies with tab bar size — read `\.tabViewBottomAccessoryPlacement` |
| `.tabBarMinimizeBehavior(_:)` | `.onScrollDown`, `.onScrollUp`, `.never`, `.automatic` |
| `ToolbarSpacer(_:)` | `.fixed` or `.flexible` spacing between toolbar items; works in customizable toolbars |
| `.backgroundExtensionEffect()` | Mirrors and blurs a view into surrounding safe area — for a detail column under a sidebar or inspector. Clips the view; use sparingly |
| `.scrollEdgeEffectStyle(_:for:)` | `.soft`, `.hard`, or `nil` for automatic |
| `.matchedTransitionSource(id:in:)` + `.navigationTransition(.zoom(sourceID:in:))` | Zoom transition from a source view into a sheet or pushed view |

**Animation and symbols**

| API | Notes |
|---|---|
| `@Animatable` | Synthesizes `Animatable` conformance and `animatableData` from a type's animatable stored properties |
| `@AnimatableIgnored` | Excludes a property from the synthesized `animatableData` |
| `.symbolEffect(.drawOn)` / `.symbolEffect(.drawOff)` | SF Symbols stroke-based reveal and dismiss |

**Other**

- Rich `TextEditor` bound to `AttributedString`, with selection change reporting.
- `.labelIconToTitleSpacing`, `.labelReservedIconWidth` for label layout.
- `WebView` / `WebPage` for embedding web content natively.

## iOS 27 additions

**Toolbars**

| API | Notes |
|---|---|
| `.visibilityPriority(_:)` | Declares which toolbar items survive as the window narrows |
| `.toolbarOverflowMenu` | Collects lower-priority items into an overflow menu |
| `.topBarPinnedTrailing` | Pins a critical action to the trailing edge |
| `.toolbarMinimizeBehavior(_:)` | Collapses the navigation bar on scroll (the toolbar counterpart to iOS 26's `tabBarMinimizeBehavior`) |

**Documents**

- `WritableDocument` / `ReadableDocument` — async and incremental disk IO, replacing the all-at-once `FileDocument` model for large documents.
- Progress reporting through Foundation's `Subprogress`.
- `DocumentCreationSource` and `NewDocumentButton` — multiple creation entry points, one button each.

**Presentation and interaction**

- Reorderable containers: drag to rearrange across `List`, `LazyVGrid`, and custom layouts. Reordering is also available on watchOS.
- `.swipeActionsContainer` — swipe actions on any `ScrollView`, not just `List` rows.
- Item-binding alerts and confirmation dialogs: presentation is driven by an optional value becoming non-`nil`, matching the `sheet(item:)` pattern.

**Performance and data flow**

- `AsyncImage` respects standard HTTP caching by default; configure it with `.asyncImageURLSession(_:)` to supply a `URLSession` with a custom `URLCache`.
- `@State` now lazily initializes class values once per view lifetime, so an expensive object in `@State` is no longer constructed on every `body` evaluation.
- `ViewBuilder` improvements, surfaced as `ContentBuilder`.

## Availability gating

Gate at the smallest scope that works, and always supply a fallback:

```swift
if #available(iOS 26, *) {
    content.glassEffect(.regular.interactive(), in: .rect(cornerRadius: 16))
} else {
    content.background(.ultraThinMaterial, in: RoundedRectangle(cornerRadius: 16))
}
```

For a modifier that has no pre-26 equivalent, a custom `ViewModifier` keeps call sites clean:

```swift
extension View {
    func glassBackground(cornerRadius: CGFloat) -> some View {
        modifier(GlassBackground(cornerRadius: cornerRadius))
    }
}
```

Flag as an issue: new API used without a gate when the deployment target is lower, a gate with no `else` branch on a visual modifier, and `@available` on a whole view when only one modifier needs it.

## Commonly hallucinated API

Reject these if they appear in code or in a suggestion — none of them exist:

| Invented | Real |
|---|---|
| `.scrollExtensionMode(.underSidebar)` | `.scrollEdgeEffectStyle(_:for:)` |
| `Glass.prominent` | `.buttonStyle(.glassProminent)` — a button style, not a glass variant |
| `.glassEffect(.thick)` / `.thin` | Only `.regular`, `.clear`, `.identity` |
| `LanguageModelSession.isAvailable` | `SystemLanguageModel.default.availability` |

When unsure whether an API exists, say so and check Apple's documentation rather than guessing a plausible-looking name.
