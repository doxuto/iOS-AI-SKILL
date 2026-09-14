# Testing with TestStore

`TestStore` proves how a feature evolves over time: every state mutation, every action an effect sends back, and the fact that no effect is left running. It is the reason to use this architecture, so a feature without tests is an unfinished feature.

## Contents

- The basic shape
- Asserting state changes
- Asserting effects
- Controlling time
- Non-exhaustive testing
- Testing navigation
- Reading test failures
- Gotchas

## The basic shape

Use Swift Testing for new tests. `TestStore` must run on the main actor.

```swift
import ComposableArchitecture
import Testing

@MainActor
struct CounterTests {
  @Test
  func increment() async {
    let store = TestStore(initialState: Counter.State()) {
      Counter()
    }

    await store.send(.incrementButtonTapped) {
      $0.count = 1
    }
  }
}
```

`await store.send(_:)` sends an action; the trailing closure describes the state **after** the action. Omit the closure when no state changes.

## Asserting state changes

Write the closure as an **absolute assignment**, not a relative mutation:

```swift
// Good — states the expected value
await store.send(.incrementButtonTapped) { $0.count = 1 }

// Bad — passes even if the starting value was wrong
await store.send(.incrementButtonTapped) { $0.count += 1 }
```

The closure receives a copy of the state before the action. If your assertion does not match what the reducer produced, the failure prints a diff of the two.

## Asserting effects

Every action an effect feeds back must be asserted with `receive`, using case key-path syntax:

```swift
@Test
func loadFact() async {
  let store = TestStore(initialState: Feature.State(count: 42)) {
    Feature()
  } withDependencies: {
    $0.factClient.fetch = { "\($0) is a great number." }
  }

  await store.send(.factButtonTapped) {
    $0.isLoading = true
  }
  await store.receive(\.factResponse.success) {
    $0.isLoading = false
    $0.fact = "42 is a great number."
  }
}
```

`\.factResponse.success` works because `@Reducer` applies `@CasePathable` to the `Action` enum.

If an effect is still running when the test finishes, the test fails with *"An effect returned for this action is still running."* That is deliberate: an unfinished effect may be about to feed wrong data into the system. Either assert on what it sends, or cancel it:

```swift
await store.send(.stopButtonTapped)  // reducer returns .cancel(id:)
// or, for a long-living effect started by a .task action:
await store.send(.task)
// ...
await store.send(.onDisappear)
await store.finish()
```

## Controlling time

Never let a test wait in real time. Override the clock:

```swift
let store = TestStore(initialState: Timer.State()) {
  Timer()
} withDependencies: {
  $0.continuousClock = ImmediateClock()
}
```

`ImmediateClock` makes every `sleep` return instantly. When the test needs to observe intermediate steps, use `TestClock` and advance it explicitly:

```swift
let clock = TestClock()
let store = TestStore(initialState: Timer.State()) {
  Timer()
} withDependencies: {
  $0.continuousClock = clock
}

await store.send(.startButtonTapped)
await clock.advance(by: .seconds(1))
await store.receive(\.timerTick) { $0.count = 1 }
await clock.advance(by: .seconds(1))
await store.receive(\.timerTick) { $0.count = 2 }
await store.send(.stopButtonTapped)
```

A `timeout:` on `receive` is a smell. It means the effect is doing real time-based work that should be behind a clock dependency.

## Non-exhaustive testing

For highly composed features, exhaustive assertions become noise. Switch the store to non-exhaustive mode and assert only what matters:

```swift
let store = TestStore(initialState: AppFeature.State()) {
  AppFeature()
}
store.exhaustivity = .off

await store.send(.tab1(.buttonTapped))
await store.receive(\.tab1.response) {
  $0.tab2.sharedCount = 1
}
```

- `.off` — skip unasserted state changes and unreceived actions silently.
- `.off(showSkippedAssertions: true)` — same, but report what was skipped as informational notes. Useful while writing the test.

Use non-exhaustive mode for integration tests at the app level. Keep leaf features exhaustive — that is where the precision pays for itself.

## Testing navigation

Presentation shows up as ordinary state:

```swift
await store.send(.editButtonTapped) {
  $0.destination = .edit(EditFeature.State(item: .mock))
}
await store.send(\.destination.edit.saveButtonTapped)
await store.receive(\.destination.edit.delegate.didSave) {
  $0.item = .updated
  $0.destination = nil
}
```

Sending an action into a presented child uses case key-path syntax on `send` as well: `store.send(\.destination.edit.saveButtonTapped)`.

When a child dismisses itself with `@Dependency(\.dismiss)`, assert the resulting `.dismiss` presentation action:

```swift
await store.send(\.destination.edit.doneButtonTapped)
await store.receive(\.destination.dismiss) {
  $0.destination = nil
}
```

## Reading test failures

| Failure | Meaning | Fix |
|---|---|---|
| "A state change does not match expectation" | Your closure and the reducer disagree | Read the printed diff; usually the assertion, sometimes the reducer |
| "An effect returned for this action is still running" | An effect never finished | Assert on its actions, cancel it, or `await store.finish()` |
| "Expected to receive an action, but received none" | Effect never sent, or real time-based asynchrony | Override the clock, or check the effect actually runs |
| "Received unexpected action" | The reducer sent something the test did not expect | Add the `receive`, or fix the reducer |
| "Unimplemented: apiClient.items" | A dependency endpoint was used but not stubbed | Stub it in `withDependencies` |
| "@Dependency(\.uuid) has no test implementation" | `uuid` used without override | `$0.uuid = .incrementing` |

## Gotchas

- Mark the test type or each test `@MainActor`. `TestStore` is main-actor isolated.
- `State` must be `Equatable` for diffing.
- A test store held at suite scope outlives individual tests and its effects bleed between them. Create the store inside each test.
- Unit test targets that link `ComposableArchitecture` statically alongside the app target can produce duplicate-symbol and dependency-isolation oddities. Link the library once, in the framework or package the tests import.
- Do not assert on internal actions that exist only to share logic — that is a design smell. See `performance.md`.
