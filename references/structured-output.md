# Structured Output

## Why @Generable, Not Free-Form Text

Never parse LLM text output with string splitting, regex, or manual JSON decoding in production code. The model's phrasing changes across OS updates and temperature settings. `@Generable` uses constrained decoding — the model physically cannot generate output that violates your Swift type.

```swift
// BAD: fragile text parsing
let response = try await session.respond(to: "List 3 tips")
let tips = response.content.split(separator: "\n").map(String.init)

// GOOD: constrained decoding guarantees the structure
@Generable
struct TipList {
    @Guide(description: "Three actionable tips", .count(3))
    var tips: [String]
}

let response = try await session.respond(to: "List 3 tips", generating: TipList.self)
let tips = response.content.tips   // Guaranteed to be exactly 3 strings
```

> The response property for structured output is `.content` — it returns the fully generated `@Generable` instance. For text-only responses, `.content` returns a `String`.

## @Guide Constraints Reference

| Constraint | Example | Effect |
|---|---|---|
| Description only | `@Guide(description: "A full name")` | Semantic hint to the model |
| `.count(n)` | `@Guide(.count(3))` | Exactly n elements in an array |
| `.maximumCount(n)` | `@Guide(.maximumCount(5))` | Up to n elements in an array |
| `.range(lo...hi)` | `@Guide(.range(1...10))` | Numeric range (Int or Double) |
| `.anyOf([])` | `@Guide(.anyOf(["PG", "PG-13", "R"]))` | Restricts output to specific string values |
| `Regex` | `@Guide(Regex { ... })` | Constrains string output via RegexBuilder |
| Combined | `@Guide(description: "Tips", .count(3))` | Semantic + structural together |

```swift
@Generable
struct MovieReview {
    @Guide(description: "A critical summary of the film")
    var summary: String

    @Guide(.anyOf(["G", "PG", "PG-13", "R"]))
    var rating: String

    @Guide(description: "Score out of 10", .range(1...10))
    var score: Int
}
```

### Regex Constraints

> Source: WWDC25 Session 301 — "Deep dive into the Foundation Models framework" demonstrates `@Guide(Regex { ... })` using Swift's `RegexBuilder` syntax.

Use Regex constraints to enforce string format patterns at the decoding level — for example, date formats, identifiers, or codes.

```swift
import RegexBuilder

@Generable
struct OrderReference {
    @Guide(description: "Order ID in format ORD-XXXX", Regex {
        "ORD-"
        Repeat(.digit, count: 4)
    })
    var orderId: String
}
```

## Supported Types for @Generable

> Source: WWDC25 Session 286 — "Meet the Foundation Models framework" covers the full list of generable types.

All of the following are generable:

- `String`
- `Int`, `Double`, `Float`, `Decimal`, `Bool`
- `Array` of any `@Generable` type
- `enum` with `@Generable` (all cases must be declared; enums with associated values are supported)
- Nested `struct` with `@Generable`
- `Optional` of any supported type (used for partial/streaming fields)
- Recursive `@Generable` types — a type may reference itself

```swift
// Recursive type — valid and supported
@Generable
struct MenuItem {
    @Guide(description: "Display name")
    var title: String

    @Guide(description: "Nested sub-items, if any")
    var children: [MenuItem]?
}
```

```swift
@Generable
enum Difficulty {
    case easy
    case moderate
    case hard
}

@Generable
struct HealthTip {
    @Guide(description: "One actionable step in a single sentence")
    var actionableStep: String

    @Guide(description: "Estimated minutes per day", .range(1...60))
    var dailyMinutes: Int

    @Guide(description: "Difficulty level for a beginner")
    var difficulty: Difficulty
}
```

## Property Ordering Affects Quality

The model generates properties in declaration order. Place fields that provide semantic context before fields that depend on it. Place summary or conclusion fields last — this lets the model "think through" the details before summarizing.

> Source: WWDC25 Session 286: "Properties are generated in the order they are declared on your Swift struct. This matters both for animations and for the quality of the model's output."

```swift
// GOOD: reasoning fields come before the conclusion
@Generable
struct TripItinerary {
    @Guide(description: "Detailed plans for each day")
    var days: [DayPlan]           // Generated first — model "thinks" through the days

    @Guide(description: "A one-sentence summary of the overall trip")
    var summary: String           // Generated last — model has full context
}

// BAD: summary generated before content it summarizes
@Generable
struct TripItinerary {
    var summary: String           // Model has no context yet
    var days: [DayPlan]
}
```

## Nested @Generable Types

Compose complex structures from smaller `@Generable` types. Each nested type must also carry the macro.

```swift
@Generable
struct DayPlan {
    @Guide(description: "Morning activity")
    var morning: String

    @Guide(description: "Afternoon activity")
    var afternoon: String

    @Guide(description: "Restaurant recommendation for dinner")
    var dinner: String
}

@Generable
struct TripItinerary {
    @Guide(description: "One plan per day", .count(3))
    var days: [DayPlan]

    @Guide(description: "Overall trip summary")
    var summary: String
}
```

## Dynamic Schemas (Runtime-Defined Structures)

Use `DynamicGenerationSchema` only when the structure cannot be known at compile time. Validate schemas before use — invalid schemas throw at construction time, not inference time. Prefer compile-time `@Generable` whenever the structure is known ahead of time.

> Source: WWDC25 Session 301 covers DynamicGenerationSchema in detail.

```swift
let riddleDynamicSchema = DynamicGenerationSchema(
    name: "Riddle",
    properties: [
        DynamicGenerationSchema.Property(name: "question", schema: DynamicGenerationSchema(type: String.self)),
        DynamicGenerationSchema.Property(name: "answers", schema: DynamicGenerationSchema(arrayOf: DynamicGenerationSchema(referenceTo: "Answer")))
    ]
)

let answerDynamicSchema = DynamicGenerationSchema(
    name: "Answer",
    properties: [
        DynamicGenerationSchema.Property(name: "text", schema: DynamicGenerationSchema(type: String.self))
    ]
)

let schema = try GenerationSchema(root: riddleDynamicSchema, dependencies: [answerDynamicSchema])
let response = try await session.respond(to: "Generate a riddle about coffee", schema: schema)

// Access fields by key — not type-safe like @Generable
let question = try response.content.value(String.self, forProperty: "question")
let answers  = try response.content.value([GeneratedContent].self, forProperty: "answers")
```
