# Navigation

TCA models navigation as **state**: a screen is presented because a piece of state is non-`nil` or because an element sits on a stack. Never call `dismiss()` on a child from a parent, and never drive navigation from a `@State` boolean in the view.

## Contents

- Tree-based navigation
- Multiple destinations with `@Reducer enum`
- The current scoping syntax
- Alerts and confirmation dialogs
- Stack-based navigation
- Dismissal
- Choosing between tree and stack

## Tree-based navigation

One optional piece of state drives one presentation.

```swift
@Reducer
struct Feature {
  @ObservableState
  struct State: Equatable {
    @Presents var editItem: EditItemFeature.State?
  }

  enum Action {
    case editButtonTapped
    case editItem(PresentationAction<EditItemFeature.Action>)
  }

  var body: some Reducer<State, Action> {
    Reduce { state, action in
      switch action {
      case .editButtonTapped:
        state.editItem = EditItemFeature.State(item: state.item)
        return .none
      case .editItem:
        return .none
      }
    }
    .ifLet(\.$editItem, action: \.editItem) {
      EditItemFeature()
    }
  }
}
```

```swift
struct FeatureView: View {
  @Bindable var store: StoreOf<Feature>

  var body: some View {
    Button("Edit") { store.send(.editButtonTapped) }
      .sheet(item: $store.scope(\.editItem, action: \.editItem)) { store in
        EditItemView(store: store)
      }
  }
}
```

`PresentationAction` has two cases you may match on: `.presented(childAction)` and `.dismiss`. `.dismiss` is sent when the user swipes the sheet away or taps outside a popover.

## Multiple destinations with `@Reducer enum`

When one screen can present several different destinations, do **not** add one optional per destination — that allows two to be non-`nil` at once. Use a single enum.

```swift
@Reducer
struct MultipleDestinations {
  @Reducer
  enum Destination {
    case drillDown(Counter)
    case popover(Counter)
    case sheet(Counter)
  }

  @ObservableState
  struct State: Equatable {
    @Presents var destination: Destination.State?
  }

  enum Action {
    case destination(PresentationAction<Destination.Action>)
    case showDrillDown
    case showPopover
    case showSheet
  }

  var body: some Reducer<State, Action> {
    Reduce { state, action in
      switch action {
      case .showDrillDown:
        state.destination = .drillDown(Counter.State())
        return .none
      case .showPopover:
        state.destination = .popover(Counter.State())
        return .none
      case .showSheet:
        state.destination = .sheet(Counter.State())
        return .none
      case .destination:
        return .none
      }
    }
    .ifLet(\.$destination, action: \.destination)
  }
}
extension MultipleDestinations.Destination.State: Equatable {}
```

Note that `.ifLet` needs no trailing reducer closure here — the `@Reducer enum` already supplies it.

## The current scoping syntax

Scope to the whole destination with the **projected** key path, then chain into the case:

```swift
struct MultipleDestinationsView: View {
  @Bindable var store: StoreOf<MultipleDestinations>

  var body: some View {
    Form {
      Button("Show drill-down") { store.send(.showDrillDown) }
      Button("Show popover") { store.send(.showPopover) }
      Button("Show sheet") { store.send(.showSheet) }
    }
    .navigationDestination(
      item: $store.scope(\.$destination, action: \.destination).drillDown
    ) { store in
      CounterView(store: store)
    }
    .popover(
      item: $store.scope(\.$destination, action: \.destination).popover
    ) { store in
      CounterView(store: store)
    }
    .sheet(
      item: $store.scope(\.$destination, action: \.destination).sheet
    ) { store in
      CounterView(store: store)
    }
  }
}
```

Two older spellings are deprecated and should be replaced on sight:

```swift
// Deprecated: TCA's own store-based modifiers
.sheet(store: store.scope(state: \.$destination.sheet, action: \.destination.sheet))

// Deprecated: non-projected scope into a specific case
.sheet(item: $store.scope(state: \.destination?.sheet, action: \.destination.sheet))

// Current
.sheet(item: $store.scope(\.$destination, action: \.destination).sheet)
```

A case that carries non-feature data must be `Identifiable` under the new scoping. An empty case produces `Optional<Void>`; drive it with `isPresented` instead:

```swift
.sheet(isPresented: Binding($store.scope(\.$destination, action: \.destination).help)) {
  HelpView()
}
```

## Alerts and confirmation dialogs

Alerts are state too, expressed with `AlertState` so they can be asserted in tests.

```swift
@ObservableState
struct State: Equatable {
  @Presents var alert: AlertState<Action.Alert>?
}

enum Action {
  case alert(PresentationAction<Alert>)
  case deleteButtonTapped

  @CasePathable
  enum Alert: Equatable {
    case confirmDeleteButtonTapped
  }
}

case .deleteButtonTapped:
  state.alert = AlertState {
    TextState("Delete this item?")
  } actions: {
    ButtonState(role: .cancel) { TextState("Cancel") }
    ButtonState(role: .destructive, action: .confirmDeleteButtonTapped) {
      TextState("Delete")
    }
  } message: {
    TextState("This cannot be undone.")
  }
  return .none

case .alert(.presented(.confirmDeleteButtonTapped)):
  // perform the delete
  return .none

case .alert:
  return .none
```

Reducer: `.ifLet(\.$alert, action: \.alert)`.
View: `.alert($store.scope(\.alert, action: \.alert))` — the **plain** key path here, not the projected one.

The equivalent for dialogs is `ConfirmationDialogState` and `.confirmationDialog($store.scope(\.confirmationDialog, action: \.confirmationDialog))`.

## Stack-based navigation

For a drill-down stack of arbitrary depth:

```swift
@Reducer
struct NavigationDemo {
  @Reducer
  enum Path {
    case screenA(ScreenA)
    case screenB(ScreenB)
    case screenC(ScreenC)
  }

  @ObservableState
  struct State: Equatable {
    var path = StackState<Path.State>()
  }

  enum Action {
    case goBackToScreen(id: StackElementID)
    case path(StackActionOf<Path>)
    case popToRoot
  }

  var body: some Reducer<State, Action> {
    Reduce { state, action in
      switch action {
      case let .goBackToScreen(id):
        state.path.pop(to: id)
        return .none

      case let .path(action):
        switch action {
        case .element(id: _, action: .screenB(.screenAButtonTapped)):
          state.path.append(.screenA(ScreenA.State()))
          return .none
        default:
          return .none
        }

      case .popToRoot:
        state.path.removeAll()
        return .none
      }
    }
    .forEach(\.path, action: \.path)
  }
}
extension NavigationDemo.Path.State: Equatable {}
```

```swift
NavigationStack(path: $store.scope(\.path, action: \.path)) {
  RootView(store: store)
} destination: { store in
  switch store.case {
  case let .screenA(store): ScreenAView(store: store)
  case let .screenB(store): ScreenBView(store: store)
  case let .screenC(store): ScreenCView(store: store)
  }
}
```

Push from a child without the parent knowing by using `NavigationLink(state:)`:

```swift
NavigationLink("Go to screen A", state: NavigationDemo.Path.State.screenA(ScreenA.State()))
```

`NavigationStackStore` is deprecated — use `NavigationStack(path: $store.scope(...))`.

## Dismissal

A child dismisses **itself** using the `dismiss` dependency. The parent does not reach in.

```swift
@Reducer
struct Child {
  @Dependency(\.dismiss) var dismiss

  // ...
  case .doneButtonTapped:
    return .run { _ in await self.dismiss() }
}
```

`await dismiss()` must be called from an effect, not directly in the reducer body. A parent can still dismiss by setting `state.destination = nil` or popping the stack, which is appropriate when the parent owns the decision.

## Choosing between tree and stack

| Use tree-based (`@Presents`) | Use stack-based (`StackState`) |
|---|---|
| Sheets, popovers, alerts, full-screen covers | A drill-down flow of arbitrary depth |
| A single, known destination from this screen | Screens that can appear repeatedly in a path |
| Deep-linking into one specific screen | Deep-linking into a whole path at once |

Stack state is a flat array, which makes constructing a deep link a matter of appending elements. Tree state nests, which makes the compiler enforce which screens can present which.
