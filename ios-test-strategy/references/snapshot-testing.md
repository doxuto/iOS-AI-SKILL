# Snapshot testing

Snapshot tests catch visual regressions that unit tests cannot see and UI tests are far too slow to cover: truncation, overflow, broken RTL layout, Dynamic Type breakage, dark-mode contrast, and design-system drift.

The standard library for this is Point-Free's [`swift-snapshot-testing`](https://github.com/pointfreeco/swift-snapshot-testing) (1.19.x). It supports both XCTest and Swift Testing.

## Contents

- Setup
- Writing a snapshot test
- The configuration matrix
- Recording and reviewing baselines
- Non-image snapshots
- Keeping a snapshot suite healthy

## Setup

```swift
// Package.swift
.package(url: "https://github.com/pointfreeco/swift-snapshot-testing", from: "1.19.0")
// ...
.testTarget(
  name: "DesignSystemTests",
  dependencies: [
    "DesignSystem",
    .product(name: "SnapshotTesting", package: "swift-snapshot-testing"),
  ]
)
```

Snapshot images are written next to the test file in `__Snapshots__/`. Commit them — they are the assertion.

## Writing a snapshot test

```swift
import SnapshotTesting
import SwiftUI
import Testing

@MainActor
struct OrderCardTests {
  @Test
  func standard() {
    let view = OrderCard(order: .fixture())
    assertSnapshot(of: view, as: .image(layout: .device(config: .iPhone13)))
  }

  @Test
  func longTitleTruncates() {
    let view = OrderCard(order: .fixture(title: String(repeating: "Very long title ", count: 10)))
    assertSnapshot(of: view, as: .image(layout: .device(config: .iPhone13)))
  }
}
```

For UIKit:

```swift
assertSnapshot(of: viewController, as: .image(on: .iPhone13))
```

Use a fixed device configuration, not "whatever simulator this runs on". A snapshot recorded on one device and verified on another fails for reasons that have nothing to do with the code.

## The configuration matrix

The value of snapshot testing comes from running the *same* view under the configurations where layout actually breaks:

```swift
@Test
func orderCardAcrossConfigurations() {
  let view = OrderCard(order: .fixture())

  assertSnapshot(
    of: view,
    as: .image(layout: .device(config: .iPhone13)),
    named: "light"
  )
  assertSnapshot(
    of: view.environment(\.colorScheme, .dark),
    as: .image(layout: .device(config: .iPhone13)),
    named: "dark"
  )
  assertSnapshot(
    of: view.environment(\.dynamicTypeSize, .accessibility3),
    as: .image(layout: .device(config: .iPhone13)),
    named: "xxxl"
  )
  assertSnapshot(
    of: view.environment(\.layoutDirection, .rightToLeft),
    as: .image(layout: .device(config: .iPhone13)),
    named: "rtl"
  )
}
```

Priorities, in order of how often they catch something real:

1. **Largest accessibility Dynamic Type size** — catches nearly every layout bug that reaches users.
2. **Longest realistic localized string** — German and Vietnamese run long; catches truncation.
3. **RTL** — if you ship Arabic or Hebrew.
4. **Dark mode.**
5. **Smallest supported device width.**

Snapshot the interesting *states*, not every view: empty, loading, error, one item, many items, longest content.

## Recording and reviewing baselines

```swift
// Record a single test
assertSnapshot(of: view, as: .image, record: true)

// Record a whole suite
withSnapshotTesting(record: .all) {
  // assertions
}
```

`record: .missing` (record only snapshots that do not exist yet) is the safest default for a suite; `.all` re-records everything and will happily bake in a regression.

Rules:

- **Never** commit a re-recorded snapshot without looking at the image diff. A snapshot test that gets re-recorded on every failure is a test that asserts nothing.
- Review snapshot changes in the PR like code. A diff of 40 images means either an intentional design-system change or a bug; both need a human.
- Record on one canonical environment. Different Xcode versions, OS versions, and Apple Silicon vs Intel can all produce sub-pixel differences that fail an otherwise-passing test. Pin the simulator and OS in the test plan, and record on the same combination CI uses.
- If cross-machine noise persists, set a small `precision` (for example `0.99`) rather than disabling the test — but treat a needed precision below ~0.98 as a sign the environment is not pinned.

## Non-image snapshots

Snapshots do not have to be images, and the text-based ones are often more useful because they diff readably in a PR:

```swift
assertSnapshot(of: order, as: .dump)                       // value structure
assertSnapshot(of: urlRequest, as: .curl)                  // outgoing request
assertSnapshot(of: try encoder.encode(order), as: .json)   // wire format
```

`.json` snapshots of encoded models are an excellent guard against accidentally changing an API contract. `.dump` is a quick way to lock in a complex value's shape.

## Keeping a snapshot suite healthy

- **Put snapshot tests in their own test target or plan.** They are slower than unit tests and their failures are a different kind of problem.
- **Budget them.** A few hundred snapshots across a design system is healthy; several thousand across every screen state is a suite nobody will maintain.
- **Fixture, don't fetch.** A snapshot of a view driven by live data changes whenever the data does.
- **Freeze anything time-dependent** — relative dates, countdowns, "new" badges based on `Date()`.
- **Delete snapshots for deleted views.** Orphaned `__Snapshots__` files accumulate silently; a periodic sweep comparing snapshot file names against test names catches them.
