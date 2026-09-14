# What to test

## Contents

- Worth testing
- Not worth testing
- Coverage
- Retrofitting tests onto untested code
- Test naming and structure

## Worth testing

- **Business rules.** Pricing, eligibility, discounts, validation, retry policy, permission logic. These are the reason the app exists and the place a bug costs real money.
- **State machines.** Anything with an enum of states and transitions between them. Test each transition, including the ones that should be rejected.
- **Boundary and edge cases.** Empty, one, many. Zero, negative, maximum. First day of month, leap year, DST transition, midnight. Nil, empty string, whitespace-only, very long, emoji, RTL text.
- **Parsing and serialization.** Decoding a real payload, decoding a payload with a missing optional field, decoding a payload with an unknown enum case, round-tripping.
- **Bug fixes.** Every fixed bug gets a test that fails before the fix. This is the highest-value test you will ever write, because you have concrete proof it catches something.
- **Concurrency-sensitive code.** Cancellation, ordering, reentrancy, actor isolation.
- **Anything you are about to refactor.** Tests written before a refactor are what make the refactor safe.

## Not worth testing

- **Generated or trivial code.** A computed property that returns a stored property. An `Equatable` conformance the compiler synthesized.
- **The framework.** That `UserDefaults` stores a value, that `Codable` decodes a correct payload with default settings, that SwiftUI renders a `Text`.
- **Implementation details.** A test that asserts "method X calls method Y" fails on every refactor and catches no user-visible bug. Assert on outcomes, not on call sequences. A spy verifying "we called the analytics client once with event `checkout_started`" is an exception — the call *is* the outcome.
- **Exact layout numbers.** `XCTAssertEqual(view.frame.height, 44)` is brittle and proves nothing. Use a snapshot test if the appearance matters.
- **Every permutation of a view's appearance.** Pick the states with real risk: longest realistic text, empty, error, loading.

## Coverage

Coverage measures which lines ran, not whether anything was asserted. A suite can reach 90% while asserting almost nothing.

Use it as a **discovery tool**, not a target:

- Low coverage on a business-rule file is a genuine finding. Investigate it.
- Low coverage on generated code, view boilerplate, or `#if DEBUG` blocks is noise. Exclude those paths from the report rather than writing tests to satisfy the number.
- A coverage gate above roughly 70–80% starts producing tests written to move a number, which are worse than no tests because they still have to be maintained.

Watch the **direction** rather than the absolute value: coverage on changed lines in a PR is a far more useful signal than project-wide coverage.

Enable coverage on the unit test plan only. Collecting it for UI tests is slow and the resulting numbers are misleading — code that ran during a UI test is not code that was verified.

## Retrofitting tests onto untested code

Do not start at the top of the file list. Prioritize:

1. **Where bugs actually come from.** Look at the last 20 bug reports and find the files behind them.
2. **Where you are about to make changes.** Tests written just before a change pay for themselves immediately.
3. **The highest-consequence code paths.** Payment, auth, data deletion, anything that loses user data.
4. **The most-changed files.** `git log --format=format: --name-only | sort | uniq -c | sort -rn | head -30` surfaces churn hotspots, which correlate strongly with defects.

For each, the usual obstacle is that the code reaches for a singleton or constructs its collaborators internally. The minimum viable change is to make those injectable — see `test-doubles.md`. Do that refactor *first*, with the existing behavior preserved, then write tests.

Resist rewriting untested code to be testable and changing its behavior in the same commit. You will not know which change broke it.

## Test naming and structure

Name a test after the behavior, so the failure message reads as a specification:

```swift
@Test func discountAppliesOnlyOncePerOrder() { }
@Test func expiredTokenTriggersRefresh() { }
@Test func emptyCartShowsPlaceholder() { }
```

Not `testDiscount()`, `test1()`, `testHappyPath()`.

Structure each test in three visible parts — arrange, act, assert — with the act step a single line. If the act step needs five lines, the setup belongs in a helper or a fixture.

```swift
@Test
func discountAppliesOnlyOncePerOrder() throws {
  // Arrange
  let order = Order.fixture(items: [.fixture(price: 100)])
  let coupon = Coupon.fixture(percentOff: 10)

  // Act
  let total = try Pricing.total(for: order, applying: [coupon, coupon])

  // Assert
  #expect(total == 90)
}
```

One logical assertion per test. Several `#expect` lines checking different facets of one outcome are fine; several unrelated scenarios in one test are not — the first failure hides the rest.

Use parameterized tests for the same behavior over many inputs rather than a loop inside one test, so each case reports separately:

```swift
@Test(arguments: [("", false), ("a@b", false), ("a@b.com", true)])
func emailValidation(input: String, isValid: Bool) {
  #expect(EmailValidator.isValid(input) == isValid)
}
```
