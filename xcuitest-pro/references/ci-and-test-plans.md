# Test plans and CI

## Contents

- Test plans
- Running from the command line
- Parallelization
- Results, screenshots, and attachments
- Keeping the suite fast
- Multi-locale and multi-device runs

## Test plans

A `.xctestplan` decouples *what runs* from *which scheme*, and is the right place to express "fast suite on every PR, full suite nightly".

A workable three-plan setup:

| Plan | Contents | Runs |
|---|---|---|
| `UnitTests.xctestplan` | All unit + integration tests | Every push |
| `SmokeUITests.xctestplan` | 5–15 critical journeys | Every PR |
| `FullUITests.xctestplan` | Everything, all locales/devices | Nightly |

Per-configuration options worth setting:

- **Environment variables and launch arguments** — set `API_STUB`, `FORCE_LOCALE`, `-DisableAnimations` here rather than hardcoding them in `setUp`, so one test body serves several configurations.
- **Test repetition** — "Retry on failure", max 1, on CI plans only.
- **Execution order** — randomized, to catch state leakage between tests.
- **Code coverage** — on for the unit plan; UI test coverage numbers are misleading and slow the run.
- **Localization / region / language** — one configuration per locale you commit to supporting.
- **Runtime sanitizers** — Main Thread Checker on; address/thread sanitizers on a separate nightly plan, since they slow execution several-fold.

Skip a test that cannot run in a given configuration rather than letting it fail:

```swift
func testAppleWatchHandoff() throws {
  try XCTSkipIf(TestConfiguration.isCI, "Requires a paired watch")
  // ...
}
```

## Running from the command line

```bash
xcodebuild test \
  -workspace MyApp.xcworkspace \
  -scheme MyApp \
  -testPlan SmokeUITests \
  -destination 'platform=iOS Simulator,name=iPhone 17,OS=27.0' \
  -resultBundlePath ./results.xcresult \
  -parallel-testing-enabled YES \
  -maximum-parallel-testing-workers 4 \
  -retry-tests-on-failure \
  -test-iterations 2
```

Split build and test so the build failure and the test failure are distinguishable, and so a build artifact can be reused across several test runs:

```bash
xcodebuild build-for-testing -scheme MyApp -destination '...' -derivedDataPath ./dd
xcodebuild test-without-building -xctestrun ./dd/Build/Products/*.xctestrun -destination '...'
```

Prepare the simulator explicitly in the CI setup step rather than trusting its state:

```bash
xcrun simctl shutdown all
xcrun simctl erase <device-udid>
xcrun simctl boot <device-udid>
xcrun simctl privacy <device-udid> grant location com.example.MyApp
xcrun simctl status_bar <device-udid> override --time 9:41 --batteryLevel 100 --cellularBars 4
```

The `status_bar` override matters if the suite includes snapshot comparisons — otherwise the clock alone changes every screenshot.

## Parallelization

Xcode parallelizes by **test class**, cloning the simulator per worker. Consequences:

- Two test classes can run simultaneously against separate app instances. Tests must not share a backend account, a fixture file, or anything else outside their own process.
- Worker count above the number of physical cores usually makes the run slower, not faster. Start at 4 and measure.
- A class with 20 tests and a class with 1 test give poor balance; split large classes.
- Use `.serialized` execution (a separate plan configuration with parallelism off) for the small set of tests that genuinely cannot be isolated, rather than turning parallelism off for everything.

## Results, screenshots, and attachments

Xcode captures a screenshot automatically at the moment of failure. Add your own for key steps:

```swift
func captureScreenshot(name: String) {
  let attachment = XCTAttachment(screenshot: app.screenshot())
  attachment.name = name
  attachment.lifetime = .keepAlways   // .deleteOnSuccess keeps bundles small
  add(attachment)
}
```

`.deleteOnSuccess` is the right default for step-by-step screenshots; `.keepAlways` for anything you want to review on green runs (such as generated App Store screenshots).

Also attach the app's log output and any stub-server transcript on failure — a UI failure is often explained by a request that returned the wrong fixture.

Archive the `.xcresult` bundle as a CI artifact. Parse it for reporting with `xcrun xcresulttool get --format json --path results.xcresult`.

## Keeping the suite fast

Budget: a PR-blocking UI suite should finish in under 10 minutes wall-clock.

- Deep-link into the screen under test instead of navigating to it.
- Inject auth instead of driving the login form.
- Disable animations.
- Delete tests that duplicate unit-test coverage. A validation rule belongs in a unit test; the UI test proves the error *appears*, not that every invalid input is caught.
- Reuse the built product across destinations with `build-for-testing` / `test-without-building`.
- Track the slowest tests over time from the `.xcresult` and treat a regression in duration as a defect.

## Multi-locale and multi-device runs

For an app shipping to several markets, do this with **test plan configurations**, not with more tests:

```
FullUITests.xctestplan
├── Configuration "en-US"   → language: en, region: US
├── Configuration "vi-VN"   → language: vi, region: VN
├── Configuration "ar-SA"   → language: ar, region: SA   (right-to-left)
└── Configuration "Large Text" → accessibility: XXXL content size
```

The same test body runs under each. This catches truncated labels, broken RTL layout, and Dynamic Type overflow — the failures that actually differ by market — without duplicating a single test. Run these nightly rather than per-PR; they multiply the suite's runtime by the number of configurations.
