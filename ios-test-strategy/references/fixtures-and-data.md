# Fixtures and test data

Bad fixtures are the quiet reason test suites become unmaintainable: every new field breaks a hundred call sites, and every test's setup buries the one value the test actually cares about.

## Contents

- The fixture pattern
- Where fixtures live
- Recorded API payloads
- Seeded databases
- Keeping fixtures honest

## The fixture pattern

Give every model one factory whose parameters all have defaults, so a test overrides only what it cares about.

```swift
#if DEBUG
extension Order {
  static func fixture(
    id: Order.ID = Order.ID(UUID(0)),
    title: String = "Test order",
    total: Decimal = 100,
    status: Status = .pending,
    items: [Item] = [.fixture()],
    placedAt: Date = Date(timeIntervalSince1970: 0)
  ) -> Order {
    Order(id: id, title: title, total: total, status: status, items: items, placedAt: placedAt)
  }
}
#endif
```

Then a test reads as the thing it is testing:

```swift
let order = Order.fixture(status: .cancelled)
```

Rules:

- **Every parameter has a default.** Adding a field to the model touches one line, not every test.
- **Defaults are deterministic.** Fixed UUIDs and fixed dates, never `UUID()` or `Date()`. A fixture that varies between runs produces snapshot churn and irreproducible failures.
- **Defaults are boring and valid.** The fixture's job is to be a valid baseline; each test supplies the interesting deviation.
- **Name the concept, not the shape:** `.fixture(status: .cancelled)` at the call site, or a named variant `Order.cancelled` when the same combination appears in many tests.
- Use `fixture` (or `mock`, or `stub` — pick one) consistently across the codebase.

For a collection, add a helper rather than repeating the array literal:

```swift
extension Array where Element == Order {
  static func fixtures(count: Int = 3) -> [Order] {
    (0..<count).map { Order.fixture(id: Order.ID(UUID($0)), title: "Order \($0)") }
  }
}
```

## Where fixtures live

- **Model fixtures** belong next to the model, behind `#if DEBUG`, so both the test target and SwiftUI previews can use them. Duplicating fixtures in each test target is how they drift.
- In a modular SPM codebase, a `<Module>TestSupport` target that depends on `<Module>` and is depended on by test targets is cleaner than `#if DEBUG` — it keeps test code out of the shipping binary entirely. See `spm-test-layout.md`.
- **JSON payloads** go in the test target's `Resources/`, loaded via `Bundle.module`.

```swift
extension Data {
  static func fixture(named name: String) throws -> Data {
    let url = try #require(Bundle.module.url(forResource: name, withExtension: "json"))
    return try Data(contentsOf: url)
  }
}
```

## Recorded API payloads

Decoding tests are only worth something if the payload is real. Record it from the actual API rather than hand-writing what you think it returns:

```bash
curl -s https://api.example.com/orders/123 | jq . > Tests/Resources/order_detail.json
```

Keep a small set per endpoint:

- `<endpoint>.json` — a full, typical response
- `<endpoint>_minimal.json` — only the required fields, all optionals absent
- `<endpoint>_unknown_enum.json` — an enum value your app does not know about
- `<endpoint>_error.json` — the error envelope

The minimal and unknown-enum cases are where decoding actually breaks in production.

Strip real user data, tokens, and identifiers before committing. Re-record when the API changes, and treat a decoding test that suddenly fails after a backend deploy as the contract test it is.

## Seeded databases

For SwiftData or Core Data, build the container in memory and seed it per test:

```swift
@MainActor
func makeContainer(seeding orders: [Order] = .fixtures()) throws -> ModelContainer {
  let container = try ModelContainer(
    for: Order.self,
    configurations: ModelConfiguration(isStoredInMemoryOnly: true)
  )
  for order in orders { container.mainContext.insert(order) }
  try container.mainContext.save()
  return container
}
```

A fresh container per test is the only reliable way to keep tests order-independent. Do not share one at suite scope.

Migration tests are the exception: they need a real store file. Copy a checked-in store from a previous schema version into a temp directory, open it with the current schema, and assert the data survived. Keep one such fixture per released schema version.

## Keeping fixtures honest

- **A fixture that no real API could produce hides bugs.** If `total` is always positive in fixtures and the API can return a negative, the test suite will never find the crash.
- **Include hostile values** in at least one fixture per model: empty string, very long string, emoji, RTL text, `nil` for every optional.
- **Do not let fixtures encode business logic.** If `Order.fixture()` computes its total from its items, a bug in that computation is now invisible to every test that uses the fixture. Pass the value in.
- **Prune.** A fixture used by one test that was deleted two years ago is dead weight; grep periodically.
