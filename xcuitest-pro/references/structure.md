# Test structure

## Contents

- The Robot (Page Object) pattern
- Naming
- setUp and tearDown
- Accessibility identifiers
- Test size and scope

## The Robot (Page Object) pattern

A UI test has two jobs that pull in opposite directions: it must read like a description of what a user does, and it must contain the brittle details of how to find elements. Separate them.

- **The test** states intent: "open the cart, assert one item, check out."
- **The robot** owns everything screen-specific: queries, taps, waits, screen-level assertions.

When a screen changes, exactly one robot changes, and every test that touches that screen keeps working.

```swift
@MainActor
struct LoginRobot {
  let app: XCUIApplication

  @discardableResult
  func enterCredentials(email: String, password: String) -> Self {
    let emailField = app.textFields[A11y.Login.email]
    XCTAssertTrue(emailField.waitForExistence(timeout: 5))
    emailField.tap()
    emailField.typeText(email)

    let passwordField = app.secureTextFields[A11y.Login.password]
    passwordField.tap()
    passwordField.typeText(password)
    return self
  }

  @discardableResult
  func submit() -> Self {
    app.buttons[A11y.Login.submit].tap()
    return self
  }

  @discardableResult
  func assertErrorShown() -> Self {
    XCTAssertTrue(app.staticTexts[A11y.Login.error].waitForExistence(timeout: 5))
    return self
  }
}
```

Returning `Self` and marking methods `@discardableResult` gives a readable chain without forcing every call to be used.

Guidelines:

- A robot never contains `XCTestCase` lifecycle code and never knows about other robots.
- A robot method either performs one user action or makes one assertion. Do not build `loginAndNavigateToSettingsAndChangeTheme()`.
- Navigation that produces a new screen returns the *next* robot, or the test constructs it. Pick one convention and keep it.
- Put robots in the UI test target, next to the tests.

## Naming

- Test class: `<Feature>UITests` — `CheckoutUITests`, `OnboardingUITests`.
- Test method: `test<Actor><Action><Outcome>` — `testUserCanCompleteCheckout`, `testInvalidPasswordShowsError`. The name should say what breaks when the test fails.
- Identifier constants: `<screen>.<element>` — `cart.checkoutButton`. A flat, dotted namespace is greppable and sorts sensibly.

## setUp and tearDown

```swift
final class CheckoutUITests: XCTestCase {
  private var app: XCUIApplication!

  override func setUp() {
    super.setUp()
    continueAfterFailure = false
    app = XCUIApplication()
    app.launchArguments += ["-UITest", "-ResetState"]
    app.launch()
  }

  override func tearDown() {
    app = nil
    super.tearDown()
  }
}
```

- `continueAfterFailure = false` is right for UI tests: once one step fails the rest of the journey is meaningless, and letting it continue produces a wall of cascading failures that hides the real one.
- Launch the app in `setUp`, not in each test, unless different tests need different launch configurations — then launch per test with a small helper.
- `XCUIApplication` is not cheap to construct repeatedly. Hold one per test case.
- Async setup uses `override func setUp() async throws`. Do not mix the async and sync overrides in one class.
- Under Swift 6, `XCTestCase` subclasses that touch `XCUIApplication` should be `@MainActor` to satisfy isolation checking.

## Accessibility identifiers

Set them in the app, in a constants file shared with the test target:

```swift
Button("Check out") { /* ... */ }
  .accessibilityIdentifier(A11y.Cart.checkoutButton)
```

- Use `accessibilityIdentifier`, not `accessibilityLabel`, for test hooks. The label is what VoiceOver reads and belongs to the user; the identifier belongs to the test. Overloading the label to make a test pass degrades accessibility.
- Ship the constants in a small module (or a file with target membership in both the app and the UI test target) so a rename fails to compile.
- Identify containers as well as controls (`cart.screen`), so a robot can assert "I am on the right screen" cheaply.
- In a `List`, set the identifier on the row content, and derive per-row identifiers from stable model IDs (`cart.row.\(item.id)`), never from index.

## Test size and scope

One journey per test. Signs a test is too big:

- It logs in, then does three unrelated things.
- Its name contains "and".
- Its failure message does not tell you which feature is broken.

A UI test that takes longer than roughly 30 seconds is usually setting up state through the UI that should be injected at launch — see `state-injection.md`.
