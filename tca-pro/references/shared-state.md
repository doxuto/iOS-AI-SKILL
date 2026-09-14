# Shared and persisted state

`@Shared` lets several features reference one piece of state without threading it through every parent, while keeping value semantics for everything else. Reach for it only when state genuinely belongs to more than one feature — a signed-in user, app settings, a cart. Ordinary parent/child state should stay plain.

## Contents

- Explicit shared state
- Persisted shared state
- Mutating shared state
- Deriving and read-only shared state
- Testing shared state
- Gotchas

## Explicit shared state

Shared with no persistence key: the parent creates it, children hold a reference to the same value.

```swift
@Reducer
struct SignUpFlow {
  @ObservableState
  struct State {
    @Shared var signUpData: SignUpData
    // ...
  }
}
```

The parent initializes the underlying value once and passes the projected value down:

```swift
let signUpData = Shared(value: SignUpData())
state.path.append(.basics(BasicsFeature.State(signUpData: signUpData)))
```

A `@Shared` property does **not** participate in `Equatable` synthesis the way a plain property does — shared state is compared by value, and two features holding the same reference will see the same value. Keep `State: Equatable` for testability.

## Persisted shared state

Attach a `SharedKey` and the value is loaded on creation and written back on every mutation.

```swift
@Shared(.appStorage("hasSeenOnboarding")) var hasSeenOnboarding = false
@Shared(.inMemory("stats")) var stats = Stats()
@Shared(.fileStorage(.documentsDirectory.appending(component: "stats.json"))) var stats = Stats()
```

| Strategy | Backing store | Use for |
|---|---|---|
| `.appStorage(_:)` | `UserDefaults` | Small scalars: flags, counters, enum raw values |
| `.inMemory(_:)` | Process memory | State shared app-wide but not persisted |
| `.fileStorage(_:)` | A file on disk | Codable models, collections |

Prefer a **type-safe key** over a string literal at each call site, so the default value and type live in one place:

```swift
extension SharedKey where Self == AppStorageKey<Bool> {
  static var hasSeenOnboarding: Self { appStorage("hasSeenOnboarding") }
}

extension SharedKey where Self == FileStorageKey<Stats> {
  static var stats: Self {
    fileStorage(.documentsDirectory.appending(component: "stats.json"))
  }
}

// Call sites:
@Shared(.hasSeenOnboarding) var hasSeenOnboarding = false
@Shared(.stats) var stats = Stats()
```

`.appStorage` supports the types `UserDefaults` natively supports (`Bool`, `Int`, `Double`, `String`, `Data`, `URL`, and `RawRepresentable` over those). Anything richer goes in `.fileStorage`.

## Mutating shared state

Shared state is mutated through the projected value with `withLock`, which serializes concurrent writes:

```swift
case .incrementButtonTapped:
  state.$count.withLock { $0 += 1 }
  return .none

case let .statsResponse(newStats):
  state.$stats.withLock { $0 = newStats }
  return .none
```

Do several related mutations inside **one** `withLock` block so observers never see a half-updated value. Reading is plain: `state.count`.

## Deriving and read-only shared state

Derive a shared reference to a sub-field with a key path on the projected value:

```swift
ChildFeature.State(topics: state.$signUpData.topics)
```

Mark state a feature must not write as `SharedReader`:

```swift
@ObservableState
struct State {
  @SharedReader(.stats) var stats = Stats()
}
```

Use `SharedReader` for anything a feature only displays. It is the cheapest way to prevent a feature from mutating state it does not own.

## Testing shared state

`TestStore` asserts on shared state like any other state:

```swift
await store.send(.incrementButtonTapped) {
  $0.$count.withLock { $0 = 1 }
}
```

Two rules that catch most problems:

- **Persistence leaks between tests.** `.appStorage` and `.fileStorage` are backed by in-memory test doubles under test, but a value written in one test can still be visible in another within the same suite if state is created at suite scope. Create shared state inside each test, or override the backing store per test.
- **Override a shared value up front** rather than driving the feature into that state:

```swift
@Test
func loggedInFlow() async {
  @Shared(.currentUser) var currentUser = User.mock
  let store = TestStore(initialState: Feature.State()) { Feature() }
  // ...
}
```

For UI tests, set persisted defaults through launch arguments rather than by reaching into the app — see the `xcuitest-pro` skill.

## Gotchas

- `@Shared` is not `Hashable`. If the enclosing state must be `Hashable`, hash the other fields.
- `@Shared` encodes and decodes the underlying value, so `Codable` conformance on the enclosing type works, but a decoded value creates a **new** reference rather than reconnecting to the original one. Persist the key, not the container.
- Never make everything shared. If the answer to "who owns this?" is "one feature", it should be plain state passed down.
- A shared mutation does not send an action, so it is invisible in the action log. Keep the number of places that mutate a given shared value small.
