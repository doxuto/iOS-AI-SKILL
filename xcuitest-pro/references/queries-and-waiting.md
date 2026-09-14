# Queries and waiting

## Contents

- How queries work
- Choosing a query
- Waiting correctly
- Interacting with elements
- Scrolling
- Text input and the keyboard
- System alerts and interruptions
- Debugging a query that finds nothing

## How queries work

`XCUIElement` is a *promise*, not a reference. Building a query costs nothing; **resolving** it does — it round-trips to the app process and snapshots the accessibility hierarchy. Every `.exists`, `.tap()`, `.label`, or `.count` is a resolution.

The practical consequences:

- Narrow the query before resolving. `app.cells.buttons["x"]` resolves less of the tree than `app.descendants(matching: .any)["x"]`.
- Store a resolved element in a `let` if you use it several times, instead of rebuilding the query.
- `app.buttons["x"].exists` is a full round trip. In a loop, that dominates the test's runtime.
- Very large hierarchies (a long unvirtualized list, a complex table) make every query slow. That is usually a sign the screen should use a lazy container.

## Choosing a query

Prefer, in order:

1. `app.buttons[A11y.Cart.checkout]` — element type + identifier. Fast and unambiguous.
2. `app.otherElements[A11y.Cart.screen].buttons[...]` — scoped to a container when identifiers repeat.
3. `app.buttons.matching(NSPredicate(format: "identifier BEGINSWITH %@", "cart.row")).element(boundBy: 0)` — when the set is dynamic.

Avoid:

- Localized labels (`app.buttons["Add to cart"]`) — breaks on copy edits and in every other locale.
- `element(boundBy:)` over an unstable ordering — position changes silently.
- `app.descendants(matching: .any)` — resolves the whole tree.
- `firstMatch` used to paper over ambiguity. It is fine as a deliberate optimization when exactly one match is expected and you want to stop the search early; it is not a fix for a query that matches several things.

Element type matters: a SwiftUI `Button` is `.buttons`, a `Text` is `.staticTexts`, a `TextField` is `.textFields`, a `SecureField` is `.secureTextFields`, a `List` row is `.cells`, a container with `accessibilityElement(children: .contain)` is `.otherElements`. Querying the wrong type finds nothing and the failure message will not say why.

## Waiting correctly

Never `sleep()`. Every wait should be expressed as a condition with a timeout.

**Existence:**

```swift
XCTAssertTrue(app.staticTexts[A11y.Confirmation.title].waitForExistence(timeout: 10))
```

**Disappearance:**

```swift
let spinner = app.activityIndicators[A11y.Cart.spinner]
let gone = NSPredicate(format: "exists == false")
expectation(for: gone, evaluatedWith: spinner)
waitForExpectations(timeout: 10)
```

**Any other property** — use a predicate expectation:

```swift
let button = app.buttons[A11y.Cart.checkout]
let enabled = NSPredicate(format: "isEnabled == true")
let exp = XCTNSPredicateExpectation(predicate: enabled, object: button)
wait(for: [exp], timeout: 10)
```

**Hittability**, which is the condition `tap()` actually needs:

```swift
let exp = XCTNSPredicateExpectation(
  predicate: NSPredicate(format: "isHittable == true"),
  object: app.buttons[A11y.Cart.checkout]
)
wait(for: [exp], timeout: 10)
```

Timeout guidance: 5 seconds for a local state change, 10 for anything involving a stubbed network, 30 for app launch on a cold CI simulator. A timeout longer than 30 seconds means the test is waiting on something it should have controlled at launch.

## Interacting with elements

- `exists` means "present in the hierarchy". `isHittable` means "can actually receive a tap". An element behind a sheet, off-screen, or covered by a keyboard exists but is not hittable — this is the most common cause of "tap did nothing".
- `tap()` on a non-hittable element fails. Scroll to it or wait for hittability first.
- `coordinate(withNormalizedOffset:).tap()` taps a position regardless of hittability. It is a last resort; it ignores exactly the conditions you want the test to verify.
- For sliders, use `adjust(toNormalizedSliderPosition:)`. For pickers, `adjust(toPickerWheelValue:)`.
- `press(forDuration:)` for long press; `swipeLeft()`/`swipeRight()` for swipe actions on rows.

## Scrolling

```swift
func scrollToElement(_ element: XCUIElement, in scrollView: XCUIElement, maxSwipes: Int = 10) {
  var swipes = 0
  while !element.isHittable && swipes < maxSwipes {
    scrollView.swipeUp()
    swipes += 1
  }
  XCTAssertTrue(element.isHittable, "Could not scroll to element after \(maxSwipes) swipes")
}
```

Bound the loop. An unbounded `while !element.exists { swipeUp() }` hangs until the test times out when the element is genuinely absent, which turns a clear assertion failure into a 10-minute mystery.

## Text input and the keyboard

```swift
let field = app.textFields[A11y.Login.email]
XCTAssertTrue(field.waitForExistence(timeout: 5))
field.tap()
field.typeText("user@example.com")
```

- `typeText` requires the field to have keyboard focus. Tap first.
- Clear a field by selecting all and typing over it, or by tapping the clear button:

```swift
field.tap()
field.press(forDuration: 1.0)
app.menuItems["Select All"].tap()
field.typeText(newValue)
```

- The software keyboard may be disabled on CI simulators (hardware keyboard connected), which changes behavior. Force it on in the simulator settings used by CI, or use `XCUIDevice`-independent input.
- Dismiss the keyboard explicitly before asserting on elements it covers.

## System alerts and interruptions

Permission dialogs (notifications, location, camera, tracking) belong to the system, not the app, and appear non-deterministically.

```swift
addUIInterruptionMonitor(withDescription: "System dialog") { alert in
  for label in ["Allow", "Allow While Using App", "OK", "Cho phép"] {
    let button = alert.buttons[label]
    if button.exists { button.tap(); return true }
  }
  return false
}
// The monitor only fires after the next interaction with the app:
app.tap()
```

Better: avoid them entirely. Do not request permissions in test mode, or pre-grant them via the simulator (`xcrun simctl privacy <device> grant <service> <bundle-id>`) in the CI setup step. A test whose subject *is* the permission primer should be the only one that lets the dialog appear.

## Debugging a query that finds nothing

1. `print(app.debugDescription)` dumps the accessibility hierarchy as the test sees it. This answers most questions immediately.
2. Check the element type — a SwiftUI view wrapped in a `Button` may surface as `.buttons`, but the same content inside a `.contentShape` + `onTapGesture` surfaces as `.otherElements` or not at all.
3. Check whether an ancestor has `accessibilityElement(children: .combine)`, which merges descendants into one element and hides their identifiers.
4. Check for `.accessibilityHidden(true)` on the element or an ancestor.
5. Use Xcode's Accessibility Inspector against the running simulator.
