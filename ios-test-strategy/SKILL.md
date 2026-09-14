---
name: ios-test-strategy
description: Decides what to test and at which layer for an iOS or macOS project, and designs the test architecture around it — the test pyramid, unit vs integration vs snapshot vs UI tests, test doubles, coverage targets, fixtures, and how to lay out test targets across a modular SPM codebase. Use when a user asks what tests to write, how to test a feature, why a suite is slow or flaky, how to raise coverage, or how to structure test targets.
license: MIT
metadata:
  author: doxuto
  version: "1.0"
---

Decide what to test, at which layer, and how to structure the test suite. This skill is about strategy; the mechanics live in three sibling skills:

- **`swift-testing-pro`** — writing unit and integration tests with Swift Testing (`@Test`, `#expect`, `#require`).
- **`xcuitest-pro`** — writing UI tests with XCTest (`XCUIApplication`, robots, launch-state injection).
- **`tca-pro`** — testing Composable Architecture features with `TestStore`.

Read this skill first when the question is *what to test*; read those when the question is *how to write it*.

## The layers

| Layer | Framework | Runs in | Speed | What it proves |
|---|---|---|---|---|
| Unit | Swift Testing | Test process | ~1ms | One type behaves correctly, including edge cases |
| Integration | Swift Testing | Test process | ~10–100ms | Several units compose correctly; the feature's logic end-to-end with stubbed I/O |
| Snapshot | Swift Testing + SnapshotTesting | Test process | ~50–500ms | A view renders as intended across sizes, locales, and appearances |
| UI | XCTest / XCUITest | Separate app process | ~5–60s | A real user journey works in the shipped app |

Swift Testing has no UI-automation or performance-measurement API. UI tests and `measure(metrics:)` performance tests stay in XCTest. Both frameworks coexist in one project.

## Target shape

Aim for a suite whose **total PR-blocking runtime is under 10 minutes**, weighted roughly:

- ~70% unit and integration tests
- ~20% snapshot tests
- ~10% UI tests

These are proportions of *effort and count*, not of value. The right absolute numbers depend on the project; the shape is what matters. A suite where UI tests dominate is slow, flaky, and gives vague failure messages. A suite with only unit tests misses wiring bugs — the ones where every part works and the app still does not.

## Deciding where a test belongs

Ask what could break, and pick the cheapest layer that would catch it:

| The risk | Test it here |
|---|---|
| A calculation, parser, formatter, validator, or reducer is wrong | Unit |
| A state machine reaches an invalid state | Unit |
| A feature's logic and effects, with I/O stubbed | Integration (`TestStore` for TCA) |
| A decoder breaks on a real API payload | Integration, using a recorded fixture |
| A migration loses data | Integration, against a real store |
| A view truncates, overflows, or breaks in RTL or Dynamic Type | Snapshot |
| A design-system component regresses visually | Snapshot |
| A critical journey is broken end-to-end in the real app | UI |
| Deep links, push handling, launch, or permissions | UI |
| Something involving third-party SDKs or the system keyboard | UI |

Two rules that resolve most arguments:

1. **Push down.** If a behavior can be proven at a lower layer, prove it there. Testing every validation rule through the UI is the single most common cause of a suite the team stops trusting.
2. **Test once.** Duplicating a rule at three layers triples maintenance and does not triple confidence.

## Workflow for a new feature

1. Write down the feature's invariants and its failure modes, in one line each.
2. Assign each to a layer using the table above.
3. Write the unit and integration tests alongside the implementation, not after.
4. Add a snapshot test for each new view state that has a visual risk (long text, empty, error, loading).
5. Add a UI test only if the feature introduces a **new critical journey**. Extending an existing journey usually means extending an existing test.
6. Check the suite still runs inside its time budget.

## References

- `references/what-to-test.md` — deciding what is worth testing, what is not, coverage targets and what they do and do not mean, and how to prioritize when retrofitting tests onto untested code.
- `references/test-doubles.md` — stubs, fakes, spies and mocks in Swift; structs of closures over protocols; controlling time, randomness, and IDs; designing for testability.
- `references/snapshot-testing.md` — `swift-snapshot-testing` setup, strategies, recording and reviewing baselines, multi-configuration matrices, and keeping snapshot suites from rotting.
- `references/fixtures-and-data.md` — fixture design, `Item.mock` conventions, recorded API payloads, seeded databases, and keeping fixtures honest.
- `references/spm-test-layout.md` — organizing test targets across a modular Swift Package Manager codebase, test-only helper modules, and what belongs in the app target's tests.
