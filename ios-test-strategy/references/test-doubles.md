# Test doubles and designing for testability

## Contents

- The four kinds of double
- Structs of closures over protocols
- Controlling time, randomness, and identity
- Making existing code testable
- Rules

## The four kinds of double

| Kind | Does | Use for |
|---|---|---|
| **Stub** | Returns canned values | Feeding the system under test a known input |
| **Fake** | A real, simplified implementation | An in-memory database or cache |
| **Spy** | Records what it was called with | Verifying a side effect happened (analytics, logging) |
| **Mock** | Asserts on calls, fails if expectations are unmet | Rarely needed; usually a spy plus an assertion is clearer |

Most tests need a stub. Reach for a spy only when the *call itself* is the observable outcome. A "mock everything" suite tests the mocks.

## Structs of closures over protocols

For a dependency interface, prefer a struct of `@Sendable` closures to a protocol:

```swift
struct APIClient {
  var items: @Sendable () async throws -> [Item]
  var save: @Sendable (Item) async throws -> Void
}
```

Why:

- A test constructs exactly the behavior it needs, inline, with no new type:

  ```swift
  let client = APIClient(
    items: { [Item.fixture()] },
    save: { _ in throw NetworkError.offline }
  )
  ```

- No parallel hierarchy of `MockAPIClient`, `StubAPIClient`, `FailingAPIClient` classes to maintain.
- Overriding one endpoint leaves the rest at their defaults: `client.items = { [] }`.
- It composes: a logging or retrying wrapper is a function from `APIClient` to `APIClient`.

A protocol is still the right choice when you need an existential in a heterogeneous collection, when a third-party API demands one, or when the interface has stored state that closures would awkwardly capture.

If the project uses the `Dependencies` library, `@DependencyClient` generates this shape and gives each endpoint an "unimplemented" default that fails the test by name when an unstubbed endpoint is called. That failure mode — loud and specific — is worth a lot; replicate it by hand otherwise:

```swift
static let unimplemented = APIClient(
  items: { Issue.record("APIClient.items was called but not stubbed"); return [] },
  save: { _ in Issue.record("APIClient.save was called but not stubbed") }
)
```

## Controlling time, randomness, and identity

Anything non-deterministic must be injected, or the test is unreliable by construction.

| Instead of | Inject |
|---|---|
| `Date()` | A `() -> Date` closure, or `@Dependency(\.date)` |
| `Task.sleep`, `Timer`, `DispatchQueue.asyncAfter` | A `Clock` — `ImmediateClock` or `TestClock` in tests |
| `UUID()` | A `() -> UUID` closure, or `@Dependency(\.uuid)` with `.incrementing` |
| `Int.random(in:)` | A seeded `RandomNumberGenerator` |
| `Locale.current`, `TimeZone.current`, `Calendar.current` | Explicit values |
| `URLSession.shared` | An injected session, or a `URLProtocol` stub |
| `UserDefaults.standard` | An injected `UserDefaults(suiteName:)` |
| `FileManager.default` | An injected client, or a temp directory per test |

```swift
let clock = TestClock()
let sut = Poller(clock: clock, fetch: { "value" })

await sut.start()
await clock.advance(by: .seconds(30))
#expect(sut.lastValue == "value")
```

`TestClock` gives explicit control over time; `ImmediateClock` makes every sleep return instantly. Use `TestClock` when intermediate states matter, `ImmediateClock` when they do not.

## Making existing code testable

The usual blockers, and the minimum change for each:

**Singleton access inside the type.**

```swift
// Before
final class Checkout {
  func submit() async throws {
    try await APIManager.shared.post("/orders", body: order)
  }
}

// After — the singleton can stay, it just moves to the boundary
final class Checkout {
  private let post: @Sendable (String, Data) async throws -> Void
  init(post: @escaping @Sendable (String, Data) async throws -> Void = APIManager.shared.post) {
    self.post = post
  }
}
```

A defaulted initializer parameter means no call site changes, so this refactor is cheap and mechanical.

**Collaborators constructed internally.** Same fix: accept them, default them.

**Static methods over global state.** Convert to an instance method on a type that holds its dependencies, keeping a static convenience wrapper if call sites are numerous.

**Logic living in a view controller or a view body.** Extract the pure part into a free function or a small type and test that. The view keeps calling it.

**A private method you want to test.** That is usually a hint that it wants to be its own type. Do not make it `internal` just for the test, and do not use `@testable import` as a substitute for a good boundary — though `@testable` is entirely appropriate for reaching internal API in the module you own.

## Rules

- A double should be as simple as the test allows. A stub that also validates arguments and tracks call counts is a second implementation you now have to debug.
- Do not assert on interactions unless the interaction is the outcome.
- Never let a double diverge silently from the real thing. If a fake database ignores a constraint the real one enforces, the tests lie. Keep fakes small, and cover the real implementation with a narrow set of integration tests.
- Give each test its own instances. A double shared at suite scope carries state between tests.
- Name doubles for their behavior in this test — `failingClient`, `emptyStore` — not `mockClient`.
