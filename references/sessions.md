# Sessions

## LanguageModelSession — Full Init Parameters

`LanguageModelSession` accepts several optional parameters at initialization. All are optional.

```swift
// Minimal
let session = LanguageModelSession()

// With instructions (developer-controlled persona)
let session = LanguageModelSession(
    instructions: "You are a concise code reviewer. Focus on Swift best practices."
)

// With a specific model adapter
let session = LanguageModelSession(
    model: SystemLanguageModel(useCase: .contentTagging)
)

// With tools registered for the session's lifetime
let session = LanguageModelSession(
    tools: [GetWeatherTool()],
    instructions: "You are a travel assistant with access to live weather data."
)

// Restored from a prior transcript (used in context window recovery — see below)
let session = LanguageModelSession(transcript: condensedTranscript)
```

### Guardrails Parameter

The `guardrails` parameter controls content safety filtering. Two options exist:

- **`.default`** — standard content safety guardrails (the default).
- **`.permissiveContentTransform`** — a less restrictive mode intended for content transformation use cases (e.g., summarizing news articles). The model's built-in safety training still applies; guardrails cannot be fully disabled.

```swift
// Use permissive guardrails for content transformation tasks
let session = LanguageModelSession(
    guardrails: .permissiveContentTransform,
    instructions: "Summarize the following news article."
)
```

> Note: The exact spelling of the permissive variant may differ between beta versions (`.permissiveContentTransform` vs `.permissiveContentTransformations`). Verify against the SDK you are targeting.

### Model Adapters

Use `SystemLanguageModel(useCase:)` to select a task-specific model adapter, or `SystemLanguageModel(adapter:)` to load a fine-tuned adapter.

```swift
// Task-specific adapter
let session = LanguageModelSession(
    model: SystemLanguageModel(useCase: .contentTagging)
)

// Fine-tuned adapter
let session = LanguageModelSession(
    model: SystemLanguageModel(adapter: myCustomAdapter)
)
```

## Instructions — String and Builder DSL

Instructions can be provided as a plain `String` or as a result-builder closure that composes multiple string segments:

```swift
// String form
let session = LanguageModelSession(instructions: "You are a recipe assistant.")

// Builder DSL — compose multiple segments
let session = LanguageModelSession {
    "You are a helpful recipe assistant."
    "When ingredients include rice, use the recipeTool to fetch rice recipes."
    "If rice is not present, generate recipes yourself."
}
```

## Instructions vs Prompts

Instructions are the developer's voice. Prompts are user input. Never mix them — the model is trained to obey instructions over prompts, which provides a layer of defense against prompt injection.

```swift
// GOOD: instructions set behavior; prompt carries user data
let session = LanguageModelSession(
    instructions: "You are a recipe assistant. Respond only with ingredient lists."
)
let response = try await session.respond(to: userQuery)

// BAD: user input embedded in instructions — prompt injection risk
let session = LanguageModelSession(
    instructions: "You are a recipe assistant. The user is: \(userName)."  // Never do this
)
```

> Source: WWDC25 Session 286 — "Meet the Foundation Models framework": "Instructions should come from you, the developer, while prompts can come from the user."

## Multi-Turn Conversations

```swift
let session = LanguageModelSession(instructions: "You are a travel assistant.")

let firstResponse = try await session.respond(to: "Plan a 3-day trip to Kyoto.")
let followUp = try await session.respond(to: "Make day 2 focus on temples.")
// Model understands "day 2" from prior context — no need to repeat

print(session.transcript)  // Inspect the full conversation history
```

## Accessing the Transcript

`session.transcript` exposes the full conversation history. `Transcript` is a `Collection` — you can iterate over its entries directly. The first entry is always the instructions.

```swift
let allEntries = session.transcript.entries
let firstEntry = allEntries.first   // Instructions entry
let lastEntry = allEntries.last     // Most recent response
```

## Guard Against Concurrent Requests

Gate on `isResponding` before issuing a new prompt in the same session. Overlapping requests on a single session produce undefined behavior.

> Source: WWDC25 Session 286 demonstrates gating a SwiftUI button on `session.isResponding`.

```swift
// BAD: may overlap requests on the same session
Button("Regenerate") {
    Task { try await generateResponse() }
}

// GOOD: disable control while session is active
Button("Regenerate") {
    Task { try await generateResponse() }
}
.disabled(session.isResponding)
```

## Context Window Recovery

Sessions have a finite context window (currently 4096 combined input + output tokens). Exceeding it throws `exceededContextWindowSize`. Recover by creating a new session initialized with a condensed transcript.

The pattern from WWDC25 Session 301 — "Deep dive into the Foundation Models framework":

```swift
func respond(to prompt: String) async throws -> String {
    do {
        return try await session.respond(to: prompt).content
    } catch LanguageModelSession.GenerationError.exceededContextWindowSize {
        session = makeRecoverySession(from: session)
        return try await session.respond(to: prompt).content
    }
}

// Carry over the first entry (instructions) + the last entry (most recent exchange).
// This preserves intent without replaying the full history.
private func makeRecoverySession(
    from old: LanguageModelSession
) -> LanguageModelSession {
    let allEntries = old.transcript.entries
    var condensedEntries = [Transcript.Entry]()

    if let firstEntry = allEntries.first {
        condensedEntries.append(firstEntry)
        if allEntries.count > 1, let lastEntry = allEntries.last {
            condensedEntries.append(lastEntry)
        }
    }

    let condensedTranscript = Transcript(entries: condensedEntries)
    return LanguageModelSession(transcript: condensedTranscript)
}
```

> See also: TN3193 — "Managing the on-device foundation model's context window" for Apple's official guidance on context window budgeting and recovery.

## Prewarming

Call `prewarm(promptPrefix:)` when you are confident a user will interact with an AI feature imminently. This preloads the model and reduces first-token latency.

> Source: WWDC25 Session 259 — "Code-along: Bring on-device AI to your app using the Foundation Models framework" demonstrates prewarming.

```swift
// In a view's .onAppear or task, when the AI feature is likely to be used
func prepareSession() {
    session.prewarm(promptPrefix: "")
}
```

Call `prewarm()` conservatively — only when user intent is clear. Unnecessary prewarming consumes device resources.

## One Session Per Feature Flow

Do not share a single `LanguageModelSession` across unrelated features. Each distinct conversation context should own its session — sharing a session causes unrelated prompts and responses to accumulate in the transcript, wasting context window budget and confusing the model.

```swift
// BAD: one global session for all features
class AppState {
    let sharedSession = LanguageModelSession()
}

// GOOD: each feature owns its session
final class RecipeService: RecipeGenerating {
    private var session = LanguageModelSession(instructions: "You are a recipe assistant.")
}

final class CodeReviewService: CodeReviewing {
    private var session = LanguageModelSession(instructions: "You are a Swift code reviewer.")
}
```
