# Dependencies

The `Dependencies` library ships with the Composable Architecture. Anything that reaches outside the reducer is a dependency: network clients, databases, file access, the clock, `UUID`, `Date`, analytics, location, notifications, keychain, feature flags.

## Contents

- Using a dependency
- Designing a dependency interface
- `@DependencyClient`
- Registering live, test, and preview values
- Overriding dependencies
- Built-in dependencies
- Rules

## Using a dependency

```swift
@Reducer
struct Feature {
  @Dependency(\.apiClient) var apiClient
  @Dependency(\.continuousClock) var clock
  @Dependency(\.uuid) var uuid
  // ...
}
```

Declare dependencies as properties on the reducer, never inside the effect closure and never as a global.

## Designing a dependency interface

Model the interface as a **struct of closures**, not a protocol. A struct of closures can be built ad hoc in a test with exactly the behavior that test needs, without declaring a mock type.

```swift
struct APIClient {
  var items: @Sendable () async throws -> [Item]
  var save: @Sendable (Item) async throws -> Void
  var delete: @Sendable (Item.ID) async throws -> Void
}
```

Keep the interface at the level of your domain, not the transport. `func items() async throws -> [Item]` is a dependency; `func request(_ urlRequest: URLRequest) async throws -> Data` is a thin wrapper that pushes all the interesting, untested logic back into the reducer.

## `@DependencyClient`

The `@DependencyClient` macro generates an initializer with defaults, and gives every endpoint an "unimplemented" default that reports a test failure when called. That is what makes unspecified endpoints fail loudly instead of silently returning empty data.

```swift
import DependenciesMacros

@DependencyClient
struct APIClient {
  var items: @Sendable () async throws -> [Item]
  var save: @Sendable (Item) async throws -> Void
}
```

Endpoints must be `@Sendable` closures. Non-throwing endpoints that return a non-`Void` value need an explicit default, since the macro cannot invent one:

```swift
@DependencyClient
struct AnalyticsClient {
  var isEnabled: @Sendable () -> Bool = { false }
  var track: @Sendable (String) -> Void
}
```

## Registering live, test, and preview values

```swift
extension APIClient: DependencyKey {
  static let liveValue = APIClient(
    items: { try await realNetworkCall() },
    save: { item in try await realSave(item) }
  )

  static let previewValue = APIClient(
    items: { Item.mocks },
    save: { _ in }
  )
  // testValue defaults to the unimplemented client from @DependencyClient.
}

extension DependencyValues {
  var apiClient: APIClient {
    get { self[APIClient.self] }
    set { self[APIClient.self] = newValue }
  }
}
```

- `liveValue` — production. Accessing it from a test fails loudly, which is the point.
- `testValue` — leave it as the unimplemented client so any endpoint a test forgot to stub fails the test by name.
- `previewValue` — friendly data for SwiftUI previews.

Put the live implementation in its own module (or at least its own file) so feature modules depend on the interface, not on `URLSession`.

## Overriding dependencies

In tests:

```swift
let store = TestStore(initialState: Feature.State()) {
  Feature()
} withDependencies: {
  $0.apiClient.items = { [Item.mock] }
  $0.continuousClock = ImmediateClock()
  $0.uuid = .incrementing
}
```

In previews:

```swift
#Preview {
  FeatureView(
    store: Store(initialState: Feature.State()) {
      Feature()
    } withDependencies: {
      $0.apiClient.items = { Item.mocks }
    }
  )
}
```

At the app entry point, to set a value for the whole app:

```swift
@main
struct MyApp: App {
  init() {
    prepareDependencies {
      $0.apiClient = .live(baseURL: Configuration.current.baseURL)
    }
  }
  // ...
}
```

To scope an override to a subtree, use `withDependencies(from:operation:)` when constructing the child store so the child inherits the parent's overrides.

## Built-in dependencies

Prefer these over their uncontrolled counterparts:

| Dependency | Replaces |
|---|---|
| `\.continuousClock`, `\.suspendingClock` | `Task.sleep`, `Timer`, `DispatchQueue.asyncAfter` |
| `\.date`, `\.date.now` | `Date()` |
| `\.calendar`, `\.timeZone`, `\.locale` | `Calendar.current`, `TimeZone.current`, `Locale.current` |
| `\.uuid` | `UUID()` |
| `\.withRandomNumberGenerator` | `Int.random(in:)` |
| `\.mainQueue` | `DispatchQueue.main` (legacy Combine code only) |
| `\.openURL` | `UIApplication.shared.open` |
| `\.defaultAppStorage`, `\.defaultFileStorage` | `UserDefaults.standard`, direct file IO |

In tests these have "unimplemented" or deterministic defaults (`uuid` fails unless overridden with `.incrementing`; clocks must be overridden or the test waits in real time).

## Rules

- A reducer that cannot be tested without a network, a real clock, or the file system has a missing dependency.
- Never reach for a singleton (`AnalyticsManager.shared`) from a reducer. Wrap it in a dependency; the singleton can stay, it just moves behind the interface.
- Do not put a dependency in `State`.
- Do not use `@Dependency` inside a SwiftUI view. Views read from the store; dependencies belong to reducers.
- When adding an endpoint to an existing client, add it to `liveValue` and `previewValue` in the same change, or previews silently break.
