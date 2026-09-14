# UIKit

TCA is SwiftUI-first, but the store is a plain observable object and drives UIKit equally well through the `UIKitNavigation` module of `swift-navigation`, which ships with the library.

## Observing state

`observe { }` re-runs its closure whenever any state it reads changes. Set it up once, in `viewDidLoad`.

```swift
import ComposableArchitecture
import UIKitNavigation

final class CounterViewController: UIViewController {
  let store: StoreOf<Counter>

  init(store: StoreOf<Counter>) {
    self.store = store
    super.init(nibName: nil, bundle: nil)
  }

  required init?(coder: NSCoder) { fatalError() }

  override func viewDidLoad() {
    super.viewDidLoad()

    let label = UILabel()
    let incrementButton = UIButton(
      type: .system,
      primaryAction: UIAction { [weak self] _ in
        self?.store.send(.incrementButtonTapped)
      }
    )
    // ... layout ...

    observe { [weak self] in
      guard let self else { return }
      label.text = "\(self.store.count)"
      incrementButton.isEnabled = !self.store.isLoading
    }
  }
}
```

Rules:

- Read **only** what this closure should react to. Reading an unrelated field makes the closure re-run for unrelated changes.
- Do not send actions from inside `observe` — that creates a feedback loop.
- Use `[weak self]` and bail out early; the closure outlives a single render.
- Wrap a state change in `withAnimation` on the send side, or use `UIView.animate` around the property assignment inside `observe`.

`store.publisher` (Combine) is deprecated. Use `observe`.

## Bindings to UIKit controls

`UIKitNavigation` adds observation-backed bindings to common controls:

```swift
observe { [weak self] in
  guard let self else { return }
  self.textField.text = self.store.query
}

textField.addAction(
  UIAction { [weak self] _ in
    self?.store.send(.queryChanged(self?.textField.text ?? ""))
  },
  for: .editingChanged
)
```

For a feature using `BindableAction`, send binding actions explicitly rather than deriving a SwiftUI `Binding`:

```swift
store.send(.binding(.set(\.query, textField.text ?? "")))
```

## Navigation

`UIKitNavigation` provides state-driven presentation and stack navigation that mirror the SwiftUI modifiers:

```swift
present(item: $store.scope(\.$destination, action: \.destination).detail) { store in
  DetailViewController(store: store)
}

navigationController?.pushViewController(
  item: $store.scope(\.$destination, action: \.destination).detail
) { store in
  DetailViewController(store: store)
}
```

The child still dismisses itself with `@Dependency(\.dismiss)`; the parent does not call `dismiss(animated:)` on it.

For a `StackState`-driven flow, use `NavigationStackController` with `$store.scope(\.path, action: \.path)`.

## Mixed UIKit/SwiftUI

Hosting a SwiftUI view from UIKit is the simplest bridge and keeps the SwiftUI side idiomatic:

```swift
let hosting = UIHostingController(rootView: FeatureView(store: store))
```

Pass the **same** store instance rather than constructing a second one — two stores means two independent state trees.

## Pre-iOS-17 targets

`@ObservableState` uses Swift's Observation framework, which requires iOS 17. On older deployment targets the library falls back to `swift-perception`: wrap view bodies in `WithPerceptionTracking { }` so state changes are observed. A missing wrapper produces a runtime warning pointing at the exact view. This applies to SwiftUI views; UIKit `observe` works on older targets without the wrapper.
