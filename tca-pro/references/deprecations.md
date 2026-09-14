# Deprecated API and modern replacements

Flag any of the following on sight in code targeting Composable Architecture 1.26+. Most of it dates from before the 1.7 observation release and still compiles, which is why it lingers in codebases and in model training data.

## Contents

- Observation-era replacements (1.7)
- Navigation modifiers
- Effect operators
- Scoping
- Test store
- The 2.0 deprecation traits

## Observation-era replacements (1.7)

| Deprecated | Replacement |
|---|---|
| `ViewStore`, `WithViewStore` | `@ObservableState` on `State`, read `store.field` directly in the view |
| `IfLetStore` | `if let store = store.scope(...)`, or the `item:`-based presentation modifiers |
| `ForEachStore` | `ForEach(store.scope(state: \.rows, action: \.rows))` |
| `SwitchStore` / `CaseLet` | `switch store.case { ... }` on a `@Reducer enum` |
| `BindingState`, `BindingViewState`, `BindingViewStore` | `@Bindable var store` + `BindableAction` + `BindingReducer()` |
| `Store.withState` | Read state directly from the store under `@ObservableState` |
| `store.publisher` (`StorePublisher`) | Observation APIs: `observe { }` in UIKit, plain property reads in SwiftUI |
| `TaskResult` | Swift's `Result` |
| `Reducer.reduce(into:action:)` called directly | `store.send(_:)` or `Effect.send` |

## Navigation modifiers

The store-based navigation modifiers are deprecated in favor of SwiftUI's own `item:`/`isPresented:` modifiers fed by a scoped binding.

| Deprecated | Replacement |
|---|---|
| `.sheet(store:)` | `.sheet(item: $store.scope(\.$destination, action: \.destination).case)` |
| `.fullScreenCover(store:)` | `.fullScreenCover(item: $store.scope(...).case)` |
| `.popover(store:)` | `.popover(item: $store.scope(...).case)` |
| `.navigationDestination(store:)` | `.navigationDestination(item: $store.scope(...).case)` |
| `.alert(store:)`, `.legacyAlert(store:)` | `.alert($store.scope(\.alert, action: \.alert))` |
| `.confirmationDialog(store:)` | `.confirmationDialog($store.scope(\.confirmationDialog, action: \.confirmationDialog))` |
| `NavigationStackStore` | `NavigationStack(path: $store.scope(\.path, action: \.path))` |
| `NavigationLinkStore` | `NavigationLink(state:)` |

Also deprecated: the non-projected scope into a specific case.

```swift
// Deprecated
.sheet(item: $store.scope(state: \.destination?.edit, action: \.destination.edit))

// Current
.sheet(item: $store.scope(\.$destination, action: \.destination).edit)
```

## Effect operators

| Deprecated | Replacement |
|---|---|
| `Effect.animation(_:)` | `await send(_:animation:)` inside `.run` |
| `Effect.transaction(_:)` | `await send(_:transaction:)` inside `.run` |
| `Effect.debounce(id:for:scheduler:)` | `clock.sleep()` + `.cancellable(id:cancelInFlight: true)` |
| `Effect.throttle(id:for:scheduler:latest:)` | Manual throttle with clock-based scheduling in `.run` |
| `Effect.concatenate` | Sequential `await`s in one `.run` |
| `Effect.map` | Construct the right action inside `.run` |

## Scoping

- `Scope(state:action:) { }` over **enum** state is deprecated. Use a `@Reducer enum`, or `.ifCaseLet(_:action:)` on a base reducer.
- Case-path-based `Scope.init(state:action:child:)` overloads are deprecated in favor of key-path syntax.

## Test store

- Nanosecond-based `timeout:` overloads on `receive` and `finish` are deprecated; pass a `Duration` (`.seconds(1)`).
- `TestStore.bindings` and `bindings(action:)` are deprecated; send binding actions directly with `store.send(\.binding.field, value)`.

## The 2.0 deprecation traits

1.25 introduced package traits that surface the deprecations coming in 2.0:

```swift
.package(
  url: "https://github.com/pointfreeco/swift-composable-architecture",
  from: "1.26.0",
  traits: [
    "ComposableArchitecture2Deprecations",
    "ComposableArchitecture2DeprecationOverloads"
  ]
)
```

- `ComposableArchitecture2Deprecations` — leave enabled permanently so new deprecations surface as warnings.
- `ComposableArchitecture2DeprecationOverloads` — enable temporarily during a migration; it adds overloads that can regress compile times.

Recommend enabling the first trait in any project that intends to reach 2.0. Hard deprecations warn regardless of traits.
