# Bindings and forms

## Contents

- Deriving a binding
- `BindableAction` and `BindingReducer`
- Reacting to a binding with `onChange`
- Read-only bindings
- When not to use a binding

## Deriving a binding

With `@ObservableState` on the state and `@Bindable` on the store, a binding to any writable field is derived directly:

```swift
struct SettingsView: View {
  @Bindable var store: StoreOf<Settings>

  var body: some View {
    Form {
      TextField("Display name", text: $store.displayName)
      Toggle("Notifications", isOn: $store.isNotificationsEnabled)
      Stepper("Steps: \(store.stepCount)", value: $store.stepCount, in: 0...100)
    }
  }
}
```

`BindingState`, `BindingViewState`, and `BindingViewStore` are all deprecated. Do not use them in new code.

For a view that receives a store but does not need bindings, `let store: StoreOf<Feature>` is enough. Use `@Bindable` only where a binding is actually derived.

## `BindableAction` and `BindingReducer`

For the binding to write back into the store, the feature's `Action` must conform to `BindableAction` and `body` must start with `BindingReducer()`.

```swift
@Reducer
struct Settings {
  @ObservableState
  struct State: Equatable {
    var displayName = ""
    var isNotificationsEnabled = false
    var stepCount = 0
    var sliderValue = 0.0
  }

  enum Action: BindableAction {
    case binding(BindingAction<State>)
    case resetButtonTapped
  }

  var body: some Reducer<State, Action> {
    BindingReducer()
    Reduce { state, action in
      switch action {
      case .binding(\.stepCount):
        state.sliderValue = .minimum(state.sliderValue, Double(state.stepCount))
        return .none

      case .binding:
        return .none

      case .resetButtonTapped:
        state = State()
        return .none
      }
    }
  }
}
```

`BindingReducer()` must come **before** your `Reduce`, so your own logic observes the already-mutated state. Matching on `\.stepCount` inside `case .binding(...)` lets you react to one specific field.

## Reacting to a binding with `onChange`

The current `onChange` operator returns an effect directly:

```swift
BindingReducer()
  .onChange(of: \.userSettings.isHapticFeedbackEnabled) { oldValue, state in
    .run { [newValue = state.userSettings.isHapticFeedbackEnabled] send in
      try await self.settingsClient.save(newValue)
    }
  }
```

The older overload that takes a reducer builder and an `(oldValue, newValue)` pair is deprecated.

## Read-only bindings

Some SwiftUI APIs demand a `Binding` for something the feature should control, such as a sheet the user can swipe away. Derive it from a scoped store rather than building it by hand:

```swift
.sheet(item: $store.scope(\.$destination, action: \.destination).detail) { store in
  DetailView(store: store)
}
```

For a `Bool`-driven presentation over an empty enum case:

```swift
.sheet(isPresented: Binding($store.scope(\.$destination, action: \.destination).help)) {
  HelpView()
}
```

Never write `Binding(get:set:)` over a store in a view body. It side-steps the action log, so the change is invisible in tests and in the debug printer.

## When not to use a binding

- **High-frequency controls.** A `Slider` bound straight to the store sends an action per pixel of drag. Hold the value in local `@State` and send one action when the drag ends. See `performance.md`.
- **Validated fields.** If a field needs validation, transformation, or debouncing, model it as an explicit action (`case emailChanged(String)`) so the reducer owns the rule.
- **Anything that triggers an effect on every keystroke.** Use an explicit action plus a debounced, cancellable effect.
