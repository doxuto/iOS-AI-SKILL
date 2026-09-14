---
name: xcuitest-pro
description: Writes, reviews, and stabilizes iOS UI tests using XCUITest. Use when reading, writing, or reviewing UI test targets, XCUIApplication, XCUIElement, XCUIElementQuery, accessibility identifiers for testing, launch arguments and environment used to control app state under test, test plans, or when a user reports flaky, slow, or failing UI tests.
license: MIT
metadata:
  author: doxuto
  version: "1.0"
  targets: "XCTest / XCUITest, Xcode 26+"
---

Write and review XCUITest code for reliability, speed, and maintainability. Report only genuine problems — do not nitpick or invent issues.

## Why this is a separate skill from Swift Testing

Swift Testing has no UI-automation surface: there is no replacement for `XCUIApplication`, `XCUIElement`, or `XCUIElementQuery`, and none for performance measurement (`XCTMetric`, `measure(metrics:)`). **UI tests and performance tests stay in XCTest.** Unit and integration tests should be written in Swift Testing — see the `swift-testing-pro` skill. The two frameworks coexist in one project without conflict.

Do not attempt to write a UI test with `@Test` and `#expect`. Use `XCTestCase`, `XCTAssert*`, and `XCTUnwrap`.

## Review process

1. Check the test's structure and naming using `references/structure.md`.
2. Check how app state is set up for the test using `references/state-injection.md`.
3. Check element queries and waiting using `references/queries-and-waiting.md`.
4. Diagnose flakiness using `references/flakiness.md`.
5. Check test plans, parallelization, and CI configuration using `references/ci-and-test-plans.md`.

If doing a partial review, load only the relevant reference files.

## Core instructions

- **A UI test must not depend on the network, on a real backend, or on data left behind by another test.** Launch the app in a controlled configuration every time. This is the single biggest determinant of whether a UI suite is worth having.
- **Never use `sleep()` or `Thread.sleep`.** Use `waitForExistence(timeout:)`, `XCTNSPredicateExpectation`, or `wait(for:timeout:)`.
- **Query by accessibility identifier, not by localized label.** Labels change with copy edits and break in every locale but one.
- Keep tests in the Robot (Page Object) pattern: the test reads as user intent, the robot owns the queries.
- One user journey per test. A test that exercises five features fails ambiguously and blocks five teams.
- Keep the UI suite small and focused on critical paths. UI tests are 100–1000× slower than unit tests; push coverage down the pyramid wherever possible (see the `ios-test-strategy` skill).
- Do not assert on pixel positions or exact frames. Assert on existence, enabled/selected state, and value.
- Use `XCTSkip` for tests that genuinely cannot run in an environment, rather than letting them fail.

## Canonical test

```swift
import XCTest

final class CheckoutUITests: XCTestCase {
  private var app: XCUIApplication!

  override func setUp() {
    super.setUp()
    continueAfterFailure = false
    app = XCUIApplication()
    app.launchArguments += ["-UITest", "-ResetState"]
    app.launchEnvironment["API_STUB"] = "checkout_happy_path"
    app.launch()
  }

  override func tearDown() {
    app = nil
    super.tearDown()
  }

  func testUserCanCompleteCheckout() throws {
    CatalogRobot(app: app)
      .waitUntilLoaded()
      .selectItem(identifier: "item_1")

    ItemDetailRobot(app: app)
      .addToCart()

    CartRobot(app: app)
      .open()
      .assertItemCount(1)
      .checkout()

    ConfirmationRobot(app: app)
      .assertVisible()
  }
}
```

And a robot:

```swift
@MainActor
struct CartRobot {
  let app: XCUIApplication

  @discardableResult
  func open() -> Self {
    app.buttons[A11y.Cart.openButton].tap()
    XCTAssertTrue(app.otherElements[A11y.Cart.screen].waitForExistence(timeout: 5))
    return self
  }

  @discardableResult
  func assertItemCount(_ count: Int) -> Self {
    XCTAssertEqual(app.cells.matching(identifier: A11y.Cart.row).count, count)
    return self
  }

  @discardableResult
  func checkout() -> Self {
    app.buttons[A11y.Cart.checkoutButton].tap()
    return self
  }
}
```

Identifiers live in one shared file, compiled into both the app and the UI test target, so a rename is a compile error rather than a runtime failure:

```swift
public enum A11y {
  public enum Cart {
    public static let screen = "cart.screen"
    public static let openButton = "cart.openButton"
    public static let row = "cart.row"
    public static let checkoutButton = "cart.checkoutButton"
  }
}
```

## Workflow for a new UI test

1. Name the user journey the test proves. If you cannot state it in one sentence, it is too big.
2. Decide the app's starting state and how to reach it deterministically — launch arguments, a stubbed network layer, or a seeded store. Never by driving the UI through a login flow the test does not care about.
3. Add accessibility identifiers to the elements the journey touches (`.accessibilityIdentifier(A11y.Cart.checkoutButton)`).
4. Write or extend the robot for each screen.
5. Write the test as a sequence of robot calls with assertions at the points that matter.
6. Run it three times in a row, and once on a cold simulator, before considering it done.

## Output format

If the user asks for a review, organize findings by file. For each issue:

1. State the file and relevant line(s).
2. Name the rule being violated.
3. Show a brief before/after code fix.

Skip files with no issues. End with a prioritized summary of the most impactful changes to make first.

If the user asks you to write or improve tests, follow the same rules but make the changes directly instead of returning a findings report.

Example output:

### CheckoutUITests.swift

**Line 22: `sleep()` instead of an explicit wait — this is the most common source of flakiness.**

```swift
// Before
app.buttons["Checkout"].tap()
sleep(2)
XCTAssertTrue(app.staticTexts["Order confirmed"].exists)

// After
app.buttons[A11y.Cart.checkoutButton].tap()
XCTAssertTrue(
  app.staticTexts[A11y.Confirmation.title].waitForExistence(timeout: 10)
)
```

**Line 31: Query by localized label — this test only passes in English.**

```swift
// Before
app.buttons["Add to cart"].tap()

// After
app.buttons[A11y.ItemDetail.addToCartButton].tap()
```

### Summary

1. **Flakiness (high):** The fixed `sleep(2)` on line 22 fails on slow CI machines and wastes two seconds on fast ones.
2. **Localization (high):** The label-based query on line 31 breaks the suite in every non-English locale.

End of example.

## References

- `references/structure.md` — the Robot/Page Object pattern, naming, `setUp`/`tearDown`, `continueAfterFailure`, shared identifier constants, and test size.
- `references/state-injection.md` — launch arguments and environment, stubbing the network at the app boundary, seeding persistence, bypassing login, and resetting state between tests.
- `references/queries-and-waiting.md` — element types and query performance, `waitForExistence`, expectations and predicates, scrolling, keyboard, system alerts and interruption monitors.
- `references/flakiness.md` — the common causes of flaky UI tests and their fixes, plus how to diagnose one from a failure log.
- `references/ci-and-test-plans.md` — `.xctestplan` configuration, parallel execution and test repetition, `xcodebuild`/`xcresult`, screenshots and attachments, and keeping the suite fast.
