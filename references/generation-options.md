# Generation Options

## The Non-Determinism Problem

By default, `LanguageModelSession` uses random sampling — the same prompt on the same device may produce different tokens each call. This is intentional for creative use cases but must be controlled deliberately.

```swift
// Default: random sampling — output varies each call
let response = try await session.respond(to: prompt)

// Greedy: deterministic within the same session state and OS version
let response = try await session.respond(
    to: prompt,
    options: GenerationOptions(sampling: .greedy)
)
```

## Sampling Mode Reference

| Mode | Deterministic? | Behavior |
|---|---|---|
| `.greedy` | Yes (within same OS + model version) | Always picks the most likely next token |
| `.random(top:seed:)` | Yes for same seed (top-k) | Picks randomly within top-k most likely tokens |
| `.random(probabilityThreshold:seed:)` | Yes for same seed (top-p / nucleus) | Picks randomly within tokens covering threshold probability |
| No seed on either random mode | No | Output varies each call |

> Source: WWDC25 Session 301 — "Deep dive into the Foundation Models framework" covers GenerationOptions, sampling modes, and temperature in detail.

## Greedy Sampling — When to Use

Use `.greedy` when output correctness matters more than variety:

- Classification tasks (sentiment, category, tags)
- Data extraction (names, dates, structured fields)
- Validation or summarization of fixed content
- Integration tests and acceptance test snapshots

```swift
@Generable
struct SentimentResult {
    @Guide(description: "Positive, negative, or neutral")
    var sentiment: Sentiment

    @Guide(description: "Confidence from 0 to 100", .range(0...100))
    var confidence: Int
}

let result = try await session.respond(
    to: "Classify the sentiment: \(userReview)",
    generating: SentimentResult.self,
    options: GenerationOptions(sampling: .greedy)
)
```

**Important:** `.greedy` is deterministic within the same OS version and model weights. An Apple model update can change greedy output. Treat this as a model drift signal, not a code regression.

## Seed-Based Sampling — When to Use

Use a seeded random mode when you want variety across different contexts but reproducibility within a single context — for example, a "tip of the day" feature.

```swift
// Daily seed: same output for the same day, different output each new day
let dayOfYear = Calendar.current.ordinality(of: .day, in: .year, for: Date()) ?? 1
let seed = UInt64(dayOfYear)

let response = try await session.respond(
    to: "Give me a health tip for today",
    generating: HealthTip.self,
    options: GenerationOptions(sampling: .random(top: 40, seed: seed))
)
```

## Temperature — Creative Control

Temperature controls how spread out the probability distribution is before sampling.

- **Lower values** (e.g., 0.5): low variance — output is more focused and predictable
- **Higher values** (e.g., 2.0): high variance — output is more diverse and creative

> Source: WWDC25 Session 301 demonstrates `0.5` (low variance) and `2.0` (high variance). Do not treat specific numeric thresholds as hard rules — experiment with your specific prompt and task.

```swift
// Low variance — focused extraction or summarization
let response = try await session.respond(
    to: prompt,
    options: GenerationOptions(temperature: 0.5)
)

// High variance — creative or brainstorming tasks
let response = try await session.respond(
    to: prompt,
    options: GenerationOptions(temperature: 2.0)
)
```

Temperature is only meaningful with random sampling — it has no effect when `sampling: .greedy` is set.

## Limiting Response Length

Use `maximumResponseTokens` to cap how many tokens the model generates. This bounds latency for open-ended prompts.

```swift
let response = try await session.respond(
    to: prompt,
    options: GenerationOptions(
        sampling: .greedy,
        maximumResponseTokens: 150
    )
)
```

Prefer `@Guide` constraints (`.count`, `.maximumCount`) over `maximumResponseTokens` for structured output — `@Guide` constraints are enforced at the decoding level, which is more reliable than a token budget.

## Passing GenerationOptions

`GenerationOptions` is passed per-call, not per-session. You can use different strategies in the same session:

```swift
// Same session, different strategies per call
let classification = try await session.respond(
    to: classificationPrompt,
    generating: Classification.self,
    options: GenerationOptions(sampling: .greedy)
)

let suggestion = try await session.respond(
    to: suggestionPrompt,
    generating: Suggestion.self,
    options: GenerationOptions(sampling: .random(top: 40), temperature: 1.0)
)
```
