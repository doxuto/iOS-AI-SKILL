---
name: tca-pro
description: Writes, reviews, and refactors Swift features built with the Composable Architecture (TCA) and the Point-Free library ecosystem — Dependencies, Sharing, Swift Navigation, Case Paths. Use when reading, writing, or reviewing code that uses @Reducer, @ObservableState, Store, StoreOf, Effect, @Dependency, @Shared, @Presents, StackState, or TestStore, or when the user mentions TCA, Composable Architecture, or Point-Free.
license: MIT
metadata:
  author: doxuto
  version: "1.0"
  targets: "Composable Architecture 1.26+, Swift 6.2+"
---

Write and review Composable Architecture code for correctness, modern API usage, testability, and adherence to project conventions. Report only genuine problems — do not nitpick or invent issues.

## Core principles

These are the design tenets the library is built around. When a choice is ambiguous, pick the option that best serves them:

1. **Compositional architecture.** Large features are built from small ones. Every feature is a reducer that can be run standalone, embedded in a parent, or tested in isolation.
2. **Value types over reference types.** State is a struct or enum. Mutations happen in one place — the reducer.
3. **Explicit dependencies.** Anything that talks to the outside world (network, disk, clock, UUID, analytics) is a `@Dependency`, never a singleton or a direct API call inside a reducer.
4. **Controlled side effects.** Effects are returned from the reducer as values, never fired imperatively.
5. **Testability by construction.** If a feature is hard to test, the design is wrong — fix the design, not the test.
6. **Concise domain modeling.** Make invalid states unrepresentable. Prefer an enum over several optional/boolean fields that can contradict each other.

## Review process

1. Check the reducer's shape, action design, and composition using `references/reducers.md`.
2. Check effects, cancellation, and async work using `references/effects.md`.
3. Check that all outside-world access is injected using `references/dependencies.md`.
4. Check navigation modeling and dismissal using `references/navigation.md`.
5. Check bindings and form state using `references/bindings.md`.
6. Check shared/persisted state using `references/shared-state.md`.
7. Check tests using `references/testing.md`.
8. Check for performance pitfalls using `references/performance.md`.
9. Flag any deprecated or pre-1.7 API using `references/deprecations.md`.
10. If the project uses UIKit, check the binding layer using `references/uikit.md`.

If doing a partial review, load only the relevant reference files.

## Core instructions

- Target Composable Architecture 1.26 or later, Swift 6.2 or later, with modern Swift concurrency.
- Every feature uses the `@Reducer` macro and `@ObservableState` on its `State`. Never hand-roll `Reducer` conformance or use `ViewStore`/`WithViewStore` in new code.
- Actions describe **what happened**, not what to do: `.saveButtonTapped`, `.factResponse(Result<String, Error>)` — not `.setLoading(true)`, `.navigateToDetail`. The reducer decides what to do.
- Never perform side effects inline in a reducer. Reducers must be pure: mutate `inout State`, return an `Effect`.
- Never call a global, a singleton, `URLSession.shared`, `Date()`, `UUID()`, or `Task.sleep` directly from a reducer or its effects. Use `@Dependency`.
- Share logic between cases with **methods on the reducer**, never by sending a synchronous action to yourself.
- Do not introduce a third-party dependency, or a `ViewModel`/`ObservableObject` layer alongside TCA, without asking first.
- One feature per file. Keep `State`, `Action`, `body`, and the view in the same file only while the feature is small; split the view out once the file exceeds roughly 200 lines.
- If the project pins an older TCA version, match the project. Say so explicitly rather than silently writing modern API that will not compile.

## Canonical feature

Use this as the starting shape for every new feature:

```swift
import ComposableArchitecture

@Reducer
struct Feature {
  @ObservableState
  struct State: Equatable {
    var count = 0
    var fact: String?
    var isLoading = false
    @Presents var alert: AlertState<Action.Alert>?
  }

  enum Action {
    case alert(PresentationAction<Alert>)
    case decrementButtonTapped
    case factButtonTapped
    case factResponse(Result<String, any Error>)
    case incrementButtonTapped

    @CasePathable
    enum Alert: Equatable {
      case retryButtonTapped
    }
  }

  @Dependency(\.factClient) var factClient

  var body: some Reducer<State, Action> {
    Reduce { state, action in
      switch action {
      case .alert(.presented(.retryButtonTapped)):
        return self.loadFact(state: &state)

      case .alert:
        return .none

      case .decrementButtonTapped:
        state.count -= 1
        return .none

      case .factButtonTapped:
        return self.loadFact(state: &state)

      case let .factResponse(.success(fact)):
        state.isLoading = false
        state.fact = fact
        return .none

      case .factResponse(.failure):
        state.isLoading = false
        state.alert = AlertState {
          TextState("Could not load a fact.")
        } actions: {
          ButtonState(role: .cancel) { TextState("OK") }
          ButtonState(action: .retryButtonTapped) { TextState("Retry") }
        }
        return .none

      case .incrementButtonTapped:
        state.count += 1
        return .none
      }
    }
    .ifLet(\.$alert, action: \.alert)
  }

  private func loadFact(state: inout State) -> Effect<Action> {
    state.isLoading = true
    return .run { [count = state.count] send in
      await send(.factResponse(Result { try await self.factClient.fetch(count) }))
    }
  }
}
```

And the view:

```swift
import SwiftUI

struct FeatureView: View {
  @Bindable var store: StoreOf<Feature>

  var body: some View {
    Form {
      Text("\(store.count)")
      Button("Decrement") { store.send(.decrementButtonTapped) }
      Button("Increment") { store.send(.incrementButtonTapped) }
      Button("Get fact") { store.send(.factButtonTapped) }
      if let fact = store.fact {
        Text(fact)
      }
    }
    .alert($store.scope(\.alert, action: \.alert))
  }
}
```

Note the asymmetry, which is easy to get wrong: the **reducer** scopes with the projected key path (`\.$alert`, `\.$destination`), the **view** scopes with the plain one (`\.alert`) for alerts and dialogs, and with the projected one (`\.$destination`) for enum destinations.

## Workflow for a new feature

1. Model `State` so invalid combinations cannot be expressed. Reach for an enum before adding a third boolean.
2. Write `Action` as a log of things that happened — user actions, then effect responses, then child/delegate actions.
3. Declare dependencies at the top of the reducer with `@Dependency`.
4. Write `body`. Handle every case explicitly; avoid `default:` so new actions surface as compile errors.
5. Attach composition operators (`.ifLet`, `.forEach`, `Scope`, `BindingReducer`) after `Reduce`.
6. Write the view with `@Bindable var store: StoreOf<Feature>` and scope to children.
7. Write a `TestStore` test before the feature is considered done. See `references/testing.md`.

## Output format

If the user asks for a review, organize findings by file. For each issue:

1. State the file and relevant line(s).
2. Name the rule being violated.
3. Show a brief before/after code fix.

Skip files with no issues. End with a prioritized summary of the most impactful changes to make first.

If the user asks you to write or improve code, follow the same rules but make the changes directly instead of returning a findings report.

Example output:

### CounterFeature.swift

**Line 24: Side effect performed inline in the reducer — the reducer is no longer pure or testable.**

```swift
// Before
case .saveButtonTapped:
  Task { try await self.apiClient.save(state.item) }
  return .none

// After
case .saveButtonTapped:
  return .run { [item = state.item] send in
    await send(.saveResponse(Result { try await self.apiClient.save(item) }))
  }
```

**Line 41: Uncontrolled time-based asynchrony makes the test suite slow and flaky.**

```swift
// Before
return .run { send in
  try await Task.sleep(for: .seconds(1))
  await send(.timerTick)
}

// After
@Dependency(\.continuousClock) var clock
// ...
return .run { send in
  try await self.clock.sleep(for: .seconds(1))
  await send(.timerTick)
}
```

### Summary

1. **Testability (high):** The inline `Task` on line 24 escapes the effect system, so `TestStore` cannot observe it and cancellation is lost.
2. **Test speed (medium):** The real `Task.sleep` on line 41 forces the test suite to wait in real time.

End of example.

## References

- `references/reducers.md` — `@Reducer`, `@ObservableState`, action design, `Scope`, `ifLet`, `forEach`, `@Reducer enum`, parent/child communication and delegate actions.
- `references/effects.md` — `.run`, `.send`, cancellation, `cancellable(id:)`, debouncing with a clock, long-living effects, and effect anti-patterns.
- `references/dependencies.md` — `@Dependency`, `@DependencyClient`, `DependencyKey`, live/test/preview values, `withDependencies`, and designing a dependency interface.
- `references/navigation.md` — tree-based navigation (`@Presents`, `PresentationAction`, `@Reducer enum Destination`), stack-based navigation (`StackState`, `StackActionOf`), dismissal, and the current scoping syntax.
- `references/bindings.md` — `@Bindable`, `BindableAction`, `BindingReducer`, the streamlined `onChange` operator, and when a binding should not be used.
- `references/shared-state.md` — `@Shared`, `SharedKey`, `withLock`, in-memory/app-storage/file-storage persistence, and testing shared state.
- `references/testing.md` — `TestStore`, exhaustive vs non-exhaustive testing, `receive(\.case)`, controlling clocks and dependencies, and common test failures and what they mean.
- `references/performance.md` — sharing logic with methods, store scoping along stored properties, high-frequency actions, and CPU-bound work.
- `references/deprecations.md` — legacy API and its modern replacement, including the 1.25 deprecations that prepare for 2.0.
- `references/uikit.md` — driving UIKit from a store with `observe`, `UIKitNavigation`, and `@Perception` for pre-iOS-17 targets.
