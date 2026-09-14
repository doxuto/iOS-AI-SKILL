# Effects

## Contents

- The two effect constructors
- Capturing state
- Errors
- Cancellation
- Debounce, throttle, and timers
- Anti-patterns

## The two effect constructors

`Effect.run` is the workhorse. It hands you an async context and a `send` function.

```swift
case .refreshButtonTapped:
  state.isLoading = true
  return .run { send in
    await send(.itemsResponse(Result { try await self.apiClient.items() }))
  }
```

`Effect.send` synchronously feeds one action back into the system. It exists for **child → parent communication** (delegate actions) and for nothing else. It is not a general-purpose "call this other case" mechanism — see `performance.md`.

`.none` means "no work to do". Return it explicitly from every case that has no effect.

## Capturing state

An effect closure is `@Sendable` and runs after the reducer returns. It cannot read `inout State`. Capture what it needs in the capture list:

```swift
return .run { [count = state.count, id = state.item.id] send in
  await send(.response(try await self.apiClient.fetch(id, count)))
}
```

Capturing the whole `state` is legal when `State` is `Sendable`, but capture the specific fields instead: it documents the effect's real inputs and avoids retaining a large value.

## Errors

`.run` takes a trailing `catch` closure. Use it for genuinely unexpected failures; route expected failures through a `Result` in the action.

```swift
return .run { send in
  try await self.apiClient.save(item)
  await send(.saveResponse(.success(())))
} catch: { error, send in
  await send(.saveResponse(.failure(error)))
}
```

If an error escapes an effect without a `catch`, the library reports an issue in debug builds. Do not swallow errors with `try?` inside an effect — that hides failures the reducer should be handling.

## Cancellation

Give any effect that can outlive its trigger an ID, and cancel it explicitly.

```swift
private enum CancelID { case search, timer }

case let .searchQueryChanged(query):
  state.query = query
  return .run { send in
    await send(.searchResponse(Result { try await self.apiClient.search(query) }))
  }
  .cancellable(id: CancelID.search, cancelInFlight: true)

case .stopButtonTapped:
  return .cancel(id: CancelID.timer)
```

- `cancelInFlight: true` is what makes a search-as-you-type feature correct: each keystroke cancels the previous request.
- Long-running effects (`for await` over a stream, timers, websockets) must be cancellable, or they leak across feature dismissals and fail tests with "an effect is still running".
- `.ifLet` and `.forEach` already cancel a child's effects on dismissal/removal. You do not need to do that by hand.
- Inside `.run`, cooperate with cancellation: check `Task.isCancelled` around long loops, and prefer `await` points that throw `CancellationError`.

Effects tied to a view's lifetime belong in a `.task` action rather than `.onAppear`, so SwiftUI cancels them when the view disappears:

```swift
.task { await store.send(.task).finish() }
```

## Debounce, throttle, and timers

Use a clock dependency plus `.cancellable`. The Combine-based `Effect.debounce`/`Effect.throttle` operators are deprecated.

```swift
@Dependency(\.continuousClock) var clock

case let .queryChanged(query):
  state.query = query
  return .run { send in
    try await self.clock.sleep(for: .milliseconds(300))
    await send(.searchResponse(Result { try await self.apiClient.search(query) }))
  }
  .cancellable(id: CancelID.search, cancelInFlight: true)
```

Timers:

```swift
case .startTimerButtonTapped:
  return .run { send in
    for await _ in self.clock.timer(interval: .seconds(1)) {
      await send(.timerTick)
    }
  }
  .cancellable(id: CancelID.timer)
```

Never use `Task.sleep`, `DispatchQueue.main.asyncAfter`, or `Timer` inside an effect. Tests would then have to wait in real time, and the suite gets slow and flaky.

## Animations

Animate from inside the effect at the point the action is sent. `Effect.animation(_:)` and `Effect.transaction(_:)` are deprecated.

```swift
return .run { send in
  let items = try await self.apiClient.items()
  await send(.itemsResponse(.success(items)), animation: .default)
}
```

## Anti-patterns

| Anti-pattern | Why it is wrong | Do instead |
|---|---|---|
| `Task { ... }` inside a reducer | Escapes the effect system: invisible to `TestStore`, never cancelled | `return .run { ... }` |
| `URLSession.shared`, `Date()`, `UUID()` in an effect | Uncontrolled, so tests are non-deterministic | Inject via `@Dependency` |
| `.run` that only sends one action synchronously | Extra hop through the action pipeline | Call a method on the reducer |
| `Effect.concatenate` to sequence work | Deprecated; sequencing is what `async`/`await` is for | One `.run` with sequential `await`s |
| `.map` on an effect | Deprecated; obscures which action is sent | Construct the right action inside `.run` |
| Uncancellable `for await` loop | Leaks across dismissal; fails tests | `.cancellable(id:)` |
| Fire-and-forget effect with no assertion | `TestStore` fails with "an effect is still running" | Assert on what it sends, or cancel it |
