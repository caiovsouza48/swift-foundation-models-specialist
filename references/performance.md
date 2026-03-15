# Performance

## Token Cost Is Latency

Every token in instructions, prompts, tool names, tool descriptions, and generated output has a direct computational cost.

> Source: WWDC25 Session 286 — "Meet the Foundation Models framework": each token in instructions and prompts adds latency before response generation starts.

| Factor | Impact | Action |
|---|---|---|
| Long instructions | Higher baseline latency on every call | Keep instructions concise |
| Long user prompts | Higher per-call latency | Truncate or summarize inputs before passing |
| Many tools registered | Model evaluates each tool on every call | Register only tools relevant to that session |
| Long generated output | Linear latency increase | Use `.count` / `.maximumCount` `@Guide` constraints |

## Prewarming

Call `prewarm(promptPrefix:)` when you are confident a user is about to interact with an AI feature. This preloads the model and caches initial state, significantly reducing first-token latency.

> Source: WWDC25 Session 259 — "Code-along: Bring on-device AI to your app using the Foundation Models framework" demonstrates prewarming in practice.

```swift
// Prewarm when the AI feature view appears — before the user taps "Generate"
.onAppear {
    session.prewarm(promptPrefix: "")
}

// Or with a prompt prefix when you know the likely first input
session.prewarm(promptPrefix: "Give me a recipe for")
```

Use prewarming conservatively — only when user intent is clearly signalled. Unnecessary prewarming wastes device resources.

### Prewarm After .modelNotReady Resolves

When availability was `.unavailable(.modelNotReady)` and then transitions to `.available`, call `prewarm()` immediately so the session is ready before the user taps. Without it, the first-token latency penalty hits at the worst moment — right after the user has been waiting for the model to download.

```swift
func checkAvailability() {
    switch SystemLanguageModel.default.availability {
    case .available:
        featureState = .available
        // Model just became ready — prewarm so first generation is fast
        session.prewarm(promptPrefix: "")
    case .unavailable(let reason):
        switch reason {
        case .modelNotReady:
            featureState = .loading
            // Schedule a retry; when this resolves to .available, prewarm() will fire
        case .deviceNotEligible:
            featureState = .permanentlyUnavailable
        case .appleIntelligenceNotEnabled:
            featureState = .requiresSetup
        @unknown default:
            featureState = .unavailable
        }
    }
}
```

A common pattern is to call `checkAvailability()` on a timer or from a `task {}` modifier that re-fires on `scenePhase` changes. The first time it returns `.available` after a `.modelNotReady` period, `prewarm()` eliminates the cold-start penalty for the user.

## Minimize @Generable Properties

During guided generation the model populates **all** properties of a `@Generable` type, regardless of whether they are used in the UI at that moment. Every property has a generation cost. Only define properties you actually need.

```swift
// BAD: unused fields still cost tokens to generate
@Generable
struct Recipe {
    var name: String
    var description: String
    var prepTimeMinutes: Int
    var cookTimeMinutes: Int
    var servings: Int
    var difficultyRating: Double   // Not used in the list view
    var historicalBackground: String  // Not shown anywhere
}

// GOOD: lean type — only what the feature needs
@Generable
struct Recipe {
    var name: String
    var description: String
    var prepTimeMinutes: Int
}
```

If different views need different fields, define multiple lean `@Generable` types for each use case rather than one large shared type.

## Streaming for Perceived Performance

Use `streamResponse` instead of `respond` for any response the user waits to read. Streaming surfaces partial results immediately, turning latency into a progressive reveal.

> Source: WWDC25 Session 286 — "Meet the Foundation Models framework" and Session 259 (code-along) both demonstrate streaming. Session 286 notes: "Get creative with SwiftUI animations and transitions to hide latency. You have an opportunity to turn a moment of waiting into one of delight."

```swift
// BAD: user sees nothing until full response is ready
let response = try await session.respond(
    to: "Write a 3-day itinerary",
    generating: Itinerary.self
)
self.itinerary = response.content

// GOOD: user sees content appear property by property
let stream = session.streamResponse(
    to: "Write a 3-day itinerary",
    generating: Itinerary.self
)

for try await partial in stream {
    self.itinerary = partial   // PartiallyGenerated — nil fields fill in progressively
}
```

## Partial Types with Streaming

`@Generable` types automatically produce a `.PartiallyGenerated` variant where every property becomes `Optional`. Properties are populated in declaration order as the model generates them.

```swift
@Generable
struct Itinerary {
    @Guide(description: "Trip title")
    var title: String      // Non-optional: generated first, appears immediately

    @Guide(description: "Day 1 plan")
    var day1: DayPlan?     // Optional: nil until generated, then appears

    @Guide(description: "Day 2 plan")
    var day2: DayPlan?

    @Guide(description: "Day 3 plan")
    var day3: DayPlan?
}

// SwiftUI state holds the partially generated type
@State private var itinerary: Itinerary.PartiallyGenerated?
```

## SwiftUI and Streaming — View Identity

Streaming arrays requires stable identity. Without it, SwiftUI re-renders the entire list on each streaming tick, causing visual flicker and wasted layout passes.

```swift
// BAD: full list re-renders on every partial update
List(itinerary?.days ?? []) { day in
    DayPlanView(plan: day)
}

// GOOD: stable id prevents unnecessary re-renders
List(itinerary?.days ?? [], id: \.id) { day in
    DayPlanView(plan: day)
}
```

## Profiling with Instruments

Use the Foundation Models Instruments template to measure:

- Time to first token
- Total generation time
- Token throughput
- Session context size over time

Profile before and after prompt changes. Reducing instruction verbosity often yields measurable latency improvements.

## Avoid Redundant Session Creation

Creating a new `LanguageModelSession` initializes the context. Reuse sessions across calls within the same feature flow to avoid the overhead of repeated setup.

```swift
// BAD: new session for every interaction — unnecessary context initialization
func generateTip() async throws -> HealthTip {
    let session = LanguageModelSession(instructions: coachInstructions)
    return try await session.respond(to: prompt, generating: HealthTip.self).content
}

// GOOD: session lives as long as the service
final class FoundationModelsHealthCoachService: HealthCoachService {
    private var session = LanguageModelSession(instructions: coachInstructions)

    func generateTip(for goal: HealthGoal) async throws -> HealthTip {
        try await session.respond(to: goal.prompt, generating: HealthTip.self).content
    }
}
```

## Availability Check Is Free

`SystemLanguageModel.default.isAvailable` and `.availability` are synchronous property reads. They do not trigger model loading. Call them at view initialization without latency concern.
