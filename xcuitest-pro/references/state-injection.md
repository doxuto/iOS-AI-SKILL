# Controlling app state under test

A UI test runs the app in a separate process, so the test cannot reach into it. Everything must be configured **at launch**. Getting this right is what separates a UI suite that is worth running from one the team eventually deletes.

## Contents

- Launch arguments and environment
- Stubbing the network
- Seeding persistence
- Bypassing login
- Resetting between tests
- Deep links
- Time, animation, and locale

## Launch arguments and environment

```swift
// Test
app.launchArguments += ["-UITest", "-ResetState", "-DisableAnimations"]
app.launchEnvironment["API_STUB"] = "checkout_happy_path"
app.launchEnvironment["SEED_USER"] = "premium"
app.launch()
```

```swift
// App
enum TestConfiguration {
  static var isUITest: Bool {
    ProcessInfo.processInfo.arguments.contains("-UITest")
  }
  static var shouldResetState: Bool {
    ProcessInfo.processInfo.arguments.contains("-ResetState")
  }
  static var apiStub: String? {
    ProcessInfo.processInfo.environment["API_STUB"]
  }
}
```

Arguments prefixed with `-` and given a following value are also picked up by `UserDefaults`, which is the shortest path to overriding a stored flag:

```swift
app.launchArguments += ["-hasSeenOnboarding", "YES"]
// The app reads UserDefaults.standard.bool(forKey: "hasSeenOnboarding") as true.
```

Keep all of this behind one `TestConfiguration` type rather than scattering `ProcessInfo` reads through the app, and make sure the test hooks compile out of, or are inert in, release builds.

## Stubbing the network

Swap the real client at composition root when the app launches in test mode. Two workable approaches:

**Stub fixtures selected by launch environment** — simplest, and keeps fixtures in the app bundle or a test resources bundle:

```swift
func makeAPIClient() -> APIClient {
  guard let stub = TestConfiguration.apiStub else { return .live }
  return .stubbed(fixtureSet: stub)
}
```

**A local HTTP server or a `URLProtocol` subclass** — better when you need per-request control, error injection, or latency simulation:

```swift
final class StubURLProtocol: URLProtocol { /* ... */ }

let configuration = URLSessionConfiguration.ephemeral
configuration.protocolClasses = [StubURLProtocol.self]
```

Whichever you pick: **no UI test should reach a real backend.** A staging server that is down or slow turns the whole suite red for reasons unrelated to the code under test, and the team learns to ignore it.

Exception: a small, explicitly-tagged smoke suite that runs against staging on a schedule (not on every PR) is a reasonable thing to have — keep it separate.

## Seeding persistence

```swift
// Test
app.launchEnvironment["SEED_FIXTURE"] = "cart_with_three_items"

// App
if TestConfiguration.shouldResetState {
  try? store.deleteAll()
}
if let fixture = ProcessInfo.processInfo.environment["SEED_FIXTURE"] {
  try? store.load(fixture: fixture)
}
```

For SwiftData or Core Data, an in-memory container in test mode is usually cleaner than deleting and reseeding a real one:

```swift
let configuration = ModelConfiguration(isStoredInMemoryOnly: TestConfiguration.isUITest)
```

## Bypassing login

Driving the login screen in every test is slow, and it makes every test depend on the auth feature. Log in once, at launch, in test mode:

```swift
app.launchEnvironment["AUTH_TOKEN"] = "test-token"
```

Keep **one** test that actually exercises the login UI. Every other test starts signed in.

## Resetting between tests

Tests must not depend on order. On launch in test mode, reset:

- `UserDefaults` for the app's suite (`UserDefaults.standard.removePersistentDomain(forName:)`)
- The keychain entries the app owns
- The database / documents directory
- Any in-memory singletons

Since each `app.launch()` starts a fresh process, doing the reset at startup is enough — there is no need for teardown logic in the test.

## Deep links

For tests that only care about one screen deep in the app, launch straight into it:

```swift
app.launchEnvironment["DEEP_LINK"] = "myapp://orders/123"
```

This turns a 40-second navigation sequence into a 3-second test and removes the dependency on every screen in between. Reserve navigating through the UI for tests whose subject *is* the navigation.

## Time, animation, and locale

```swift
app.launchArguments += ["-DisableAnimations"]
app.launchEnvironment["FIXED_DATE"] = "2026-01-01T00:00:00Z"
app.launchEnvironment["FORCE_LOCALE"] = "vi_VN"
```

- Disabling animations (`UIView.setAnimationsEnabled(false)`, or reducing durations, in test mode) cuts several seconds per test and removes a large class of timing flakiness. Keep one test with animations on if a transition is itself under test.
- Freeze the clock for anything that displays relative dates ("2 hours ago"), or the assertion will drift.
- For a multi-locale app, run the critical-path suite under each supported locale in a separate test plan configuration rather than writing locale-specific tests. See `ci-and-test-plans.md`.
