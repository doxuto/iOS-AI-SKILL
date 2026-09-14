---
name: on-device-ai
description: Builds iOS 26+ features with Apple's Foundation Models framework for on-device AI. Use when implementing LanguageModelSession, @Generable structured output, @Guide constraints, the Tool protocol for function calling, streaming partial responses, session transcripts, or any on-device LLM integration, and when checking Apple Intelligence availability before showing an intelligence feature.
license: MIT
metadata:
  author: 199 Biotechnologies
  maintainer: doxuto
  version: "2.0"
  targets: "FoundationModels, iOS 26+"
---

# On-device AI with Foundation Models

Build features on Apple's on-device language model — no network, no API key, no per-token cost, and the user's data never leaves the device. The trade-off is a small model with a limited context window, so the design work is in choosing tasks it can actually do.

## Requirements

- iOS 26+, iPadOS 26+, macOS 26+, visionOS 26+ (watchOS 27+)
- A device that supports Apple Intelligence, with it enabled, in a supported region
- `import FoundationModels`

The model is updated with the OS, and its behavior changes between versions (26.0–26.3, 26.4, and 27.0 are distinct versions). Prompts tuned against one version should be re-checked against a newer one.

## Check availability first

This is the most commonly wrong part of Foundation Models code. There is no `LanguageModelSession.isAvailable`.

```swift
import FoundationModels
import SwiftUI

struct GenerativeView: View {
    private var model = SystemLanguageModel.default

    var body: some View {
        switch model.availability {
        case .available:
            IntelligenceView()
        case .unavailable(.deviceNotEligible):
            FallbackView(reason: "This device doesn't support Apple Intelligence.")
        case .unavailable(.appleIntelligenceNotEnabled):
            FallbackView(reason: "Turn on Apple Intelligence in Settings to use this.")
        case .unavailable(.modelNotReady):
            FallbackView(reason: "The model is still downloading. Try again shortly.")
        case .unavailable(let other):
            FallbackView(reason: "Unavailable: \(other)")
        }
    }
}
```

`SystemLanguageModel` is `Observable`, so a SwiftUI view re-renders when availability changes — for example once the model finishes downloading. `model.isAvailable` is a convenience Bool, but switching over `availability` is what lets you tell the user *why* the feature is missing.

Never ship an intelligence feature without a designed fallback. A large share of installed devices cannot run it.

## Core API

### Text generation

```swift
let session = LanguageModelSession(
    instructions: "You summarize articles in three short bullet points. Be concise and factual."
)
let response = try await session.respond(to: "…article text…")
print(response.content)   // String
```

`instructions` set the session's persistent role and are separate from the per-request prompt. Put the behavior in `instructions` and the data in the prompt — never interpolate untrusted user text into `instructions`.

A session holds a conversation. Reuse it for follow-up turns; create a new one for an unrelated task.

### Streaming

`streamResponse` yields **cumulative snapshots**, not deltas. Each element is the whole response so far, so bind it directly to the UI rather than appending.

```swift
for try await snapshot in session.streamResponse(to: prompt) {
    self.text = snapshot.content
}
```

Streaming for a `@Generable` type yields `PartiallyGenerated` values whose fields fill in progressively — which lets you render a form as it is produced.

### Structured output with `@Generable`

```swift
@Generable
struct RecipeSuggestion {
    @Guide(description: "A short, appetizing dish name")
    var name: String

    @Guide(description: "Ingredients with quantities", .count(3...10))
    var ingredients: [String]

    @Guide(description: "Total cooking time in minutes", .range(5...180))
    var cookingTimeMinutes: Int

    var difficulty: Difficulty

    @Generable
    enum Difficulty: String {
        case easy, medium, hard
    }
}

let response = try await session.respond(
    to: "Suggest a quick pasta recipe",
    generating: RecipeSuggestion.self
)
let recipe = response.content   // RecipeSuggestion
```

Two things that are easy to get wrong:

- `respond(to:generating:)` returns a `Response<Content>`. Read `.content` — it does not return the value directly.
- `@Guide` is a **property** macro that constrains one field. It is not applied to the type. Type-level description goes in `@Generable(description:)`.

Guided generation is constrained decoding: the framework forces the output to match the schema, so you do not need to parse or validate the shape. Use it instead of asking for JSON in the prompt.

Available `@Guide` constraints include `.range(_:)` for numbers, `.count(_:)` for arrays, `.anyOf(_:)` for string choices, and `.pattern(_:)` for a regex.

### Tool calling

A tool lets the model pull in data it cannot know. Note the exact shape — `Arguments` is a nested `@Generable` type, and the method is `call(arguments:)`.

```swift
struct FindContacts: Tool {
    let name = "findContacts"
    let description = "Finds a specific number of contacts"

    @Generable
    struct Arguments {
        @Guide(description: "The number of contacts to get", .range(1...10))
        let count: Int
    }

    func call(arguments: Arguments) async throws -> [String] {
        var contacts: [CNContact] = []
        // Fetch contacts using arguments.count
        return contacts.map { "\($0.givenName) \($0.familyName)" }
    }
}

let session = LanguageModelSession(tools: [FindContacts()])
let response = try await session.respond(to: "Who are my first three contacts?")
```

- `Tool` must be `Sendable` — the framework may run tools concurrently.
- `Output` is any `PromptRepresentable`, typically `String` or a `@Generable` type.
- Tool definitions are injected into the prompt and consume context window. Keep the set small and the descriptions tight.
- You own the tool's lifetime, so a tool may hold state between calls (for example, records already returned).

## Errors and limits

Handle these explicitly — they are normal operating conditions, not edge cases:

```swift
do {
    let response = try await session.respond(to: prompt)
} catch let error as LanguageModelSession.GenerationError {
    switch error {
    case .exceededContextWindowSize:
        // Start a fresh session, optionally seeded with a summary of the old transcript
    case .guardrailViolation:
        // Input or output was flagged; show a neutral message, do not retry verbatim
    case .unsupportedLanguageOrLocale:
        // Check model.supportsLocale(_:) before offering the feature
    default:
        // Fall back to a non-AI path
    }
}
```

Context window management:

- `SystemLanguageModel.default.contextSize` reports the limit; `tokenCount(for:)` measures instructions.
- A long conversation will hit the limit. When it does, create a new session and carry over a condensed summary rather than the full `Transcript`.
- `session.isResponding` guards against sending a second prompt while one is in flight.
- `session.prewarm()` loads the model ahead of the first request — call it when the user is about to need it (opening the screen), not at launch.

## Best practices

- Check `availability` before offering the feature, and design the fallback first.
- Use `@Generable` for anything structured. Never ask for JSON in the prompt and parse it yourself.
- Keep prompts short and specific. This is a small on-device model; it does not reward elaborate prompting.
- Put the role in `instructions`, the data in the prompt. Treat user text as data, never as instructions.
- Stream anything longer than a sentence — perceived latency dominates the experience.
- Use `SystemLanguageModel(useCase: .contentTagging)` for tagging and classification rather than prompting the base model for it.
- Check `supportsLocale(_:)` before offering the feature in a market whose language the model does not support.
- Write a fallback path for every call site, not just for the unavailable case.

## When not to use it

- Complex multi-step reasoning, long-document analysis, or code generation — use a server-side model.
- Anything requiring current information from the internet; the model is offline and its knowledge is fixed at the OS version.
- Image, audio, or video generation — this is text-only.
- Anything where a wrong answer is unacceptable without a human check. Treat output as a draft.

## Resources

- [Apple: Foundation Models framework](https://developer.apple.com/documentation/foundationmodels)
- [Apple: Tool protocol](https://developer.apple.com/documentation/foundationmodels/tool)
- [Apple: Managing the context window](https://developer.apple.com/documentation/foundationmodels/managing-the-context-window)
- [Apple: Updating prompts for new model versions](https://developer.apple.com/documentation/foundationmodels/updating-prompts-for-new-model-versions)
