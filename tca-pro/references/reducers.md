# Reducers, state, and actions

## Contents

- The `@Reducer` macro
- Modeling state
- Designing actions
- Composing reducers
- Parent/child communication and delegate actions
- `@Reducer enum`

## The `@Reducer` macro

Every feature is a struct annotated with `@Reducer`. The macro:

- conforms the type to `Reducer`,
- applies `@CasePathable` to the nested `Action` enum, which is what makes `\.factResponse` key-path syntax work in `receive`, `scope`, and `ifLet`,
- applies `@CasePathable` and `@dynamicMemberLookup` to a nested `State` enum if there is one,
- wires up `@Presents`-annotated properties.

```swift
@Reducer
struct Feature {
  @ObservableState
  struct State: Equatable { /* ... */ }
  enum Action { /* ... */ }
  var body: some Reducer<State, Action> { /* ... */ }
}
```

`@ObservableState` is required for the view to observe state with SwiftUI's observation. Without it the view will not re-render. It is not optional in modern TCA.

Never write `func reduce(into:action:) -> Effect<Action>` and `var body` in the same feature. Use `body` — `reduce` exists only for the rare leaf feature that needs no composition, and it cannot be composed with operators.

## Modeling state

Make invalid states unrepresentable. This is the single highest-leverage habit in TCA.

```swift
// Avoid — four booleans encode 16 states, of which 4 are valid.
struct State {
  var isLoading = false
  var results: [Item] = []
  var errorMessage: String?
  var isEmpty = false
}

// Prefer
struct State: Equatable {
  var status: Status = .idle

  enum Status: Equatable {
    case idle
    case loading
    case loaded([Item])
    case failed(String)
  }
}
```

Rules:

- `State` should be `Equatable` so `TestStore` can diff it. If a stored value is not `Equatable`, that is usually a sign it belongs in a dependency, not in state.
- Computed properties on `State` are fine and encouraged for view-facing derivations, but keep them cheap — see `performance.md` for why expensive computed properties hurt when scoped over.
- Do not store dependencies, view objects, tasks, or cancellables in `State`.
- Use `IdentifiedArrayOf<Child.State>` rather than `[Child.State]` for collections of child features, so `forEach` can address elements by stable ID.

## Designing actions

Actions are a log of events, not commands.

```swift
enum Action {
  // 1. What the user did
  case refreshButtonTapped
  case rowTapped(id: Item.ID)

  // 2. What the outside world said back
  case itemsResponse(Result<[Item], any Error>)

  // 3. Child feature actions
  case destination(PresentationAction<Destination.Action>)

  // 4. Messages this feature sends to its parent
  case delegate(Delegate)

  @CasePathable
  enum Delegate: Equatable {
    case itemSelected(Item)
  }
}
```

- Name user actions after the UI event: `saveButtonTapped`, `textFieldChanged(String)`, `onAppear`, `task`.
- Name effect responses `<thing>Response`. Carry `Result` so failure is handled in the reducer, not swallowed in the effect.
- Do **not** add actions whose only purpose is to set a piece of state (`setLoading(Bool)`, `setTitle(String)`). That moves logic out of the reducer and into the caller.
- Do **not** add actions whose purpose is to share logic between other actions — use a method. See `performance.md`.
- Handle every case explicitly in the `switch`. Avoid `default:` so that adding an action produces a compile error at every reducer that must consider it.

`TaskResult` is deprecated. Use Swift's `Result` with a typed or existential error:

```swift
case factResponse(Result<String, any Error>)
// ...
return .run { send in
  await send(.factResponse(Result { try await self.factClient.fetch(count) }))
}
```

## Composing reducers

`body` is a reducer builder. Order matters: reducers run top to bottom for a given action.

```swift
var body: some Reducer<State, Action> {
  BindingReducer()          // runs first, applies the binding mutation

  Scope(state: \.tab1, action: \.tab1) {
    Tab1Feature()
  }

  Reduce { state, action in  // then your own logic observes the result
    // ...
  }
  .ifLet(\.$destination, action: \.destination)
  .forEach(\.rows, action: \.rows) {
    RowFeature()
  }
}
```

| Operator | Use for |
|---|---|
| `Scope(state:action:)` | A child feature whose state is **always** present (a tab, a section). |
| `.ifLet(\.$child, action: \.child)` | An optional child driven by `@Presents` — sheets, alerts, drill-downs. |
| `.ifCaseLet(\.case, action: \.case)` | A child embedded in a case of an enum `State`. |
| `.forEach(\.rows, action: \.rows)` | A collection of children in an `IdentifiedArray`, or a `StackState`. |

`.ifLet` and `.forEach` must be attached to the reducer that owns the parent logic (typically the `Reduce`), and they automatically cancel a child's in-flight effects when the child is dismissed or removed. That automatic cancellation is a major reason to use them rather than hand-rolling optional handling.

## Parent/child communication

**Parent observing child:** pattern-match on the child action in the parent's reducer.

```swift
case .destination(.presented(.edit(.saveButtonTapped))):
  // react
```

Reaching deep into a child's internal actions couples the parent to the child's implementation. For anything other than a trivial case, add a `delegate` action to the child instead:

```swift
// Child
enum Action {
  case saveButtonTapped
  case delegate(Delegate)
  @CasePathable enum Delegate: Equatable { case didSave(Item) }
}

case .saveButtonTapped:
  return .send(.delegate(.didSave(state.item)))

case .delegate:
  return .none  // the child never handles its own delegate actions
```

```swift
// Parent
case let .destination(.presented(.edit(.delegate(.didSave(item))))):
  state.items[id: item.id] = item
  return .none
```

**Parent driving child:** prefer extracting shared logic into a method both can call. Sending `.send(.child(.refresh))` works but costs an extra trip through the whole action pipeline.

## `@Reducer enum`

When one slot can hold several different features — a destination or a stack path — model it as a `@Reducer enum`. The macro synthesizes the nested `State` and `Action` enums for you.

```swift
@Reducer
enum Destination {
  case detail(DetailFeature)
  case settings(SettingsFeature)
}

// Conformances the synthesized State needs must be declared in an extension:
extension Destination.State: Equatable {}
```

If a case holds plain data rather than a feature (for example `AlertState`), annotate it `@ReducerCaseIgnored` and declare the `Action` enum explicitly:

```swift
@Reducer
enum Destination {
  @ReducerCaseIgnored
  case alert(AlertState<Alert>)
  case settings(SettingsFeature)

  @CasePathable
  enum Action {
    case alert(Alert)
    case settings(SettingsFeature.Action)
  }

  enum Alert: Equatable { case confirmDeleteButtonTapped }
}
```
