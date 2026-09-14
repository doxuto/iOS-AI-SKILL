# Features added in Swift 6.3

**Requires Swift 6.3 or later.** These are newer than most model training data, so they are easy to miss when reviewing or writing tests.

## Contents

- Issue severity
- Test cancellation
- Image attachments

## Issue severity

`Issue.record` now takes a `severity`, so a test can surface a warning without failing.

```swift
Issue.record("Fixture data is older than the current API contract", severity: .warning)
```

- `.error` (the default) fails the test.
- `.warning` is reported in the results but the test still passes.

Use `.warning` for conditions worth surfacing that do not invalidate the result — a stale fixture, a deprecated path taken, a skipped optimization. It is a better tool than a `print` that nobody sees, and better than failing a test for something that is not a defect.

Do not use `.warning` to downgrade a real failure. If the assertion matters, let it fail.

## Test cancellation

`Test.cancel()` stops the current test from inside the test body.

```swift
@Test
func syncsWithRemote() async throws {
  guard Environment.hasNetworkAccess else {
    try Test.cancel("No network access in this environment")
  }
  // ...
}
```

How this differs from the alternatives:

| | Result | Use when |
|---|---|---|
| `try Test.cancel()` | Test is cancelled, reported as cancelled | The test cannot meaningfully run right now |
| `withKnownIssue { }` | Test passes, the known failure is recorded | The failure is expected and tracked |
| `ConditionTrait` (`.enabled(if:)`) | Test never starts | The condition is known before the test body runs |

Prefer a `ConditionTrait` when the condition is knowable up front — it avoids starting the test at all. Use `Test.cancel()` when the condition only becomes clear partway through.

## Image attachments

Attachments now accept images on Apple platforms and Windows, so a failing test can carry the rendering that caused it.

```swift
@Test
func chartRendersCorrectly() throws {
  let image = ChartRenderer.render(data: .fixture())
  Attachment.record(image, named: "chart.png")
  #expect(image.size == CGSize(width: 320, height: 240))
}
```

This is most useful in snapshot-style and rendering tests, where the failure message alone ("images differ") tells you nothing. Attaching both the expected and the actual image turns a mystery into a two-second diagnosis.

Keep attachment volume in check on CI — attaching an image from every test inflates the result bundle. Attach on failure, or only from tests where the image is the subject.
