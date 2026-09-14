# Test layout in a modular SPM codebase

## Contents

- The shape
- Test support targets
- What belongs in the app target's tests
- Dependency rules
- Practical problems

## The shape

One test target per feature module, mirroring the source layout:

```
Package.swift
Sources/
├── OrdersFeature/
├── OrdersClient/            ← dependency interface
├── OrdersClientLive/        ← live implementation
├── DesignSystem/
└── SharedModels/
Tests/
├── OrdersFeatureTests/
├── OrdersClientLiveTests/
└── DesignSystemTests/       ← snapshot tests live here
```

Why one per module rather than one big `AppTests`:

- Tests run against the smallest unit that can build, so an incremental test run is fast.
- A test target can only reach what its module can reach, which enforces the module boundaries.
- Failures point at a module, not at "the app".

The Xcode app target keeps its own test targets for UI tests and for anything that needs the real app bundle.

## Test support targets

Fixtures, stub clients, and helpers shared by several test targets go in a dedicated target, not behind `#if DEBUG` in the shipping module:

```swift
.target(
  name: "SharedModels"
),
.target(
  name: "SharedModelsTestSupport",
  dependencies: ["SharedModels"]
),
.testTarget(
  name: "OrdersFeatureTests",
  dependencies: ["OrdersFeature", "SharedModelsTestSupport"]
)
```

Advantages over `#if DEBUG`:

- Test code never ships, even in a debug build of the app.
- Several test targets share one definition, so fixtures cannot drift.
- SwiftUI previews can depend on it too, so previews and tests use the same data.

Name them consistently: `<Module>TestSupport`.

## What belongs in the app target's tests

Keep the app target's test targets thin. They hold only what genuinely needs the assembled app:

- UI tests (`XCUIApplication` needs a real app to launch).
- Launch, deep link, and URL-scheme handling.
- App-delegate / scene-lifecycle behavior.
- Integration tests that wire live implementations together.

Everything else moves down into a module test target, where it runs faster and in isolation.

## Dependency rules

- **A test target depends on its module, plus test support targets. Nothing else.** If `OrdersFeatureTests` needs `PaymentsFeature`, either the production dependency is real and should be declared in `Sources`, or the test is testing too much.
- **Feature modules depend on client *interfaces*, never on `*Live`.** Only the app composition root depends on the live implementations. This is what lets a feature's tests run without a network stack linked in.
- **A test-support target must not depend on a `*Live` target** — that drags the real implementation into every test target that touches fixtures.
- **Never import `@testable` across modules.** `@testable import` is for reaching internal API in the module you own. If a test needs internal API from another module, the boundary is wrong.

## Practical problems

**Duplicate symbols / duplicated library state.** Linking a library (for example `ComposableArchitecture`) statically into both the app target and a test target produces duplicate copies, which shows up as dependency overrides not taking effect, or as odd `@Dependency` failures. Link the library once, in the package or framework the tests import.

**Slow whole-package test runs.** Use test plans to scope what runs when: a plan per area for local work, an "all" plan on CI. `swift test --filter OrdersFeatureTests` works for command-line runs of the package alone.

**Resources.** A test target that loads JSON fixtures needs them declared:

```swift
.testTarget(
  name: "OrdersClientLiveTests",
  dependencies: ["OrdersClientLive"],
  resources: [.process("Resources")]
)
```

Access them with `Bundle.module`. A `Bundle(for: SomeClass.self)` lookup will not find package resources.

**Snapshot targets.** Keep snapshot tests in the module that owns the views (usually `DesignSystem` and each feature module), with their `__Snapshots__` directories committed alongside. Give them their own test plan so a snapshot failure is visibly a different class of problem from a logic failure.

**Parallelism.** Module test targets parallelize well because they are independent by construction. This is a second reason to push tests down out of the app target, where the UI tests serialize on simulator clones.
