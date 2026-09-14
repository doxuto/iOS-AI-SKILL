# Diagnosing and fixing flaky UI tests

A flaky test is worse than no test: it trains the team to re-run the suite instead of reading it. Treat a flake as a bug with a root cause, not as noise to be retried away.

## Contents

- Causes, ranked
- Diagnosing from a failure
- The quarantine rule
- Retries

## Causes, ranked

**1. Fixed sleeps.** `sleep(2)` is too short on a loaded CI machine and wasted time on a fast one.

```swift
// Fix
XCTAssertTrue(element.waitForExistence(timeout: 10))
```

**2. Shared state between tests.** A test passes alone and fails in the suite, or fails only when run second. The app is carrying over defaults, database rows, keychain entries, or a cached session.

Fix: reset everything at launch in test mode (`state-injection.md`). Verify by running the suite in randomized order.

**3. Real network.** Latency spikes, rate limits, staging deploys, and expired data all surface as UI test failures.

Fix: stub at the app boundary. No PR-blocking UI test should hit a real server.

**4. Animations.** Tapping mid-transition hits the wrong element or nothing at all.

Fix: disable animations in test mode; wait for `isHittable` rather than `exists` before tapping.

**5. Waiting on the wrong condition.** `exists` is true while the element is still off-screen, behind a sheet, or covered by the keyboard.

Fix: wait on `isHittable` for anything you are about to tap.

**6. System dialogs.** Permission alerts appear at unpredictable moments and steal the tap.

Fix: pre-grant permissions in CI setup, or do not request them in test mode. Interruption monitors are a fallback, and note that a monitor only fires after the next interaction with the app.

**7. Index-based and label-based queries.** `element(boundBy: 2)` breaks when order changes; `app.buttons["Save"]` breaks on a copy edit or in another locale.

Fix: stable accessibility identifiers derived from model IDs.

**8. Unbounded loops.** `while !element.exists { app.swipeUp() }` hangs for the full test timeout when the element is genuinely missing.

Fix: bound the loop and assert afterwards with a message.

**9. Device and OS differences.** A test that passes on iPhone 17 Pro fails on iPhone SE because the element is below the fold, or fails on iPad because the layout is a split view.

Fix: pin the simulator in the test plan; scroll explicitly rather than assuming visibility; branch on `UIDevice.current.userInterfaceIdiom` in the robot, not in the test.

**10. Parallel execution collisions.** Tests sharing a simulator clone, a fixture file, or a backend account.

Fix: make each test's state self-contained. If two tests genuinely cannot run concurrently, put them in a serialized test plan configuration rather than disabling parallelism globally.

**11. Time-dependent assertions.** "2 hours ago", "Today", anything crossing midnight or a timezone boundary.

Fix: freeze the clock and the timezone at launch.

## Diagnosing from a failure

1. **Read the failure message and the automatic screenshot.** Xcode attaches a screenshot at the point of failure; it usually shows immediately whether the app was on the wrong screen, showing an error, or mid-animation.
2. **Check the `.xcresult` bundle** for the full activity log — it shows every query, its duration, and what it resolved to.
3. **Reproduce with repetition:** `xcodebuild test -test-iterations 20 -run-tests-until-failure -only-testing:UITests/CheckoutUITests/testUserCanCompleteCheckout`. A test that fails 1 in 20 locally will fail constantly on CI.
4. **Run it second.** `-only-testing` with another test first will expose state leakage.
5. **Run it on a cold simulator.** Erase the simulator first; first-launch behavior (onboarding, permissions, empty caches) differs.
6. **Add `print(app.debugDescription)`** at the failure point to see the hierarchy as the test saw it.

## The quarantine rule

When a test flakes and cannot be fixed the same day:

1. Move it out of the PR-blocking plan into a quarantine configuration that still runs and reports, but does not block merges.
2. File a bug with the failure log attached.
3. Set a deadline. A quarantined test that no one fixes within a sprint should be deleted — an ignored red signal is worse than an absent one.

Never fix a flake by deleting the assertion or widening the timeout to 60 seconds. Both convert a visible problem into an invisible one.

## Retries

`-retry-tests-on-failure` and the test plan's "Automatically retry" setting exist, and are reasonable as a **safety net on CI** so a single infrastructure blip does not fail a build. They are not a fix. Two rules:

- Keep the retry count at 1, so a genuinely broken test still fails fast.
- Report retried-but-passed tests somewhere visible. A test that needs a retry is flaky, and if the retry hides that, the flake rate grows until the suite is meaningless.
