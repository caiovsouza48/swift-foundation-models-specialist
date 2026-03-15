# Tool Calling

## When to Use Tools

Tool calling lets the model autonomously invoke your Swift code during inference. Use it to bridge the gap between the model's language capabilities and live data it cannot know:

- Real-time data: current weather, user's calendar, live prices
- Device data: contacts, health records, location
- App-specific data: user's saved items, account balance, order history

The model decides when to call tools and how many times within a single `respond` call. All processing happens on-device.

> Source: WWDC25 Session 286 — "Meet the Foundation Models framework": "Tool calling allows the model to identify that a task may require additional information or actions and autonomously make decisions about what tool to use and when."

## Implementing the Tool Protocol

The `Tool` protocol requires a `name`, `description`, `Arguments` type (must be `@Generable`), and a `call(arguments:)` method. The protocol inherits from `Sendable`.

```swift
import FoundationModels

struct GetWeatherTool: Tool {
    let name = "getWeather"
    let description = "Retrieve the current weather for a given city. Use when the user asks about weather conditions."

    @Generable
    struct Arguments {
        @Guide(description: "The city name, e.g. 'Tokyo' or 'New York'")
        var city: String
    }

    func call(arguments: Arguments) async throws -> ToolOutput {
        let places = try await CLGeocoder().geocodeAddressString(arguments.city)
        let weather = try await WeatherService.shared.weather(for: places.first!.location!)
        let temp = weather.currentWeather.temperature.value

        return ToolOutput("\(arguments.city)'s temperature is \(temp) degrees.")
    }
}
```

> **Concurrency note:** The `Tool` protocol requires `Sendable` conformance. For struct-based tools with no mutable state, this is automatic. For class-based stateful tools (see below), you must handle concurrent access to shared state yourself.

## Tool with Context

The `Tool` protocol also supports a `call(arguments:context:)` variant that provides a `ToolCallContext` — this gives the tool read access to the parent session's transcript, enabling tools that reason about conversation history.

```swift
func call(arguments: Arguments, context: ToolCallContext) async throws -> ToolOutput {
    // context provides access to the session's transcript
    let history = context.transcript
    // Use conversation history to inform the tool's response
    ...
}
```

Use this variant when a tool needs conversation context to make better decisions. For simple data-fetching tools, the standard `call(arguments:)` is sufficient.

## Tool Naming Rules

Tool names are presented to the model as part of the prompt — they consume tokens and affect whether the model calls the tool correctly.

- **No spaces**: `"getWeather"` ✓ — `"get weather"` ✗
- **No punctuation**: `"findContact"` ✓ — `"find-contact"` ✗
- **Short**: prefer single camelCase words or short compound words
- **Description is the trigger**: write the description as a clear, unambiguous sentence about when to call the tool

```swift
// BAD: long name with spaces, vague description
let name = "get current weather information for a city"
let description = "This tool can be used to get weather info when needed."

// GOOD: concise name, precise trigger condition in description
let name = "getWeather"
let description = "Fetch current weather for a city. Call when the user asks about weather or temperature."
```

> Source: WWDC25 Session 286: "Keep tool names short and descriptions concise — they reduce token overhead."

## Registering Tools with a Session

Tools must be passed at session initialization. They are available for the session's entire lifetime and cannot be added or removed afterwards.

```swift
let session = LanguageModelSession(
    tools: [GetWeatherTool(), FindContactTool()],
    instructions: "You are a helpful personal assistant. Use tools to look up real-time information."
)

let response = try await session.respond(to: "What's the weather in Tokyo right now?")
// Model autonomously calls GetWeatherTool, then formulates the final response
```

## Arguments Must Be @Generable

Tool arguments are generated via constrained decoding — the same mechanism as `@Generable` response types. If `Arguments` is not `@Generable`, the tool will not compile.

> Source: WWDC25 Session 286: "The reason your arguments need to be generable is because tool calling is built on guided generation to ensure that the model will never produce invalid tool names or arguments."

```swift
@Generable
struct Arguments {
    @Guide(description: "Search query string")
    var query: String

    @Guide(description: "Maximum number of results", .range(1...10))
    var limit: Int
}
```

## Stateful Tools — Use a Class

The model may call the same tool multiple times within a single request, potentially in parallel. If your tool needs to maintain state across calls (e.g., tracking which items have already been picked), use a `class`, not a `struct`, and handle concurrent access carefully.

> Source: WWDC25 Session 301 — "Deep dive into the Foundation Models framework" demonstrates this pattern with a `FindContactTool`.

```swift
class FindContactTool: Tool {
    let name = "findContact"
    let description = "Finds a contact from a specified age generation."

    var pickedContacts = Set<String>()  // Mutable state shared across calls

    @Generable
    struct Arguments {
        let generation: Generation
    }

    func call(arguments: Arguments) async throws -> ToolOutput {
        // Tools may execute in parallel — guard shared mutable state appropriately
        var contacts = fetchContacts(for: arguments.generation)
        contacts.removeAll(where: { pickedContacts.contains($0.givenName) })

        guard let picked = contacts.randomElement() else {
            return ToolOutput("Could not find a contact.")
        }

        pickedContacts.insert(picked.givenName)
        return ToolOutput(picked.givenName)
    }
}
```

If your tool has no state, a `struct` is fine and is naturally safe for parallel calls.

## ToolOutput

`ToolOutput` is constructed from a `String` — a natural-language description of the result that the model reads and integrates into its response.

```swift
return ToolOutput("The temperature in Tokyo is 22°C.")
```

## ToolCallError

Tool-specific failures are reported via `LanguageModelSession.ToolCallError`, which carries a reference to the failing `tool` and the `underlyingError`. Handle these separately from `GenerationError` when you need to distinguish tool failures from generation failures.

## Parallel Calls — Stateless Is Safe

If your tool is stateless, it is naturally safe for parallel invocations:

```swift
struct SearchTool: Tool {
    let name = "search"
    let description = "Search the app catalog for items matching a query."

    @Generable
    struct Arguments {
        var query: String
    }

    func call(arguments: Arguments) async throws -> ToolOutput {
        let results = try await Catalog.shared.search(query: arguments.query)
        return ToolOutput(results.map(\.title).joined(separator: ", "))
    }
}
```

## Privacy Considerations

Tools that access sensitive user data (contacts, health, location) require explicit authorization. Check authorization status before accessing protected data — a tool call does not bypass system permission requirements.

```swift
func call(arguments: Arguments) async throws -> ToolOutput {
    guard ContactStore.authorizationStatus == .authorized else {
        return ToolOutput("Contacts access not authorized.")
    }
    let contact = try await ContactStore.shared.find(name: arguments.name)
    return ToolOutput(contact.fullName)
}
```

## Tool Calling vs @Generable — Decision Table

| Need | Use |
|---|---|
| Structured output generated from the prompt | `@Generable` |
| Live or real-time data the model cannot know | Tool calling |
| Multi-step reasoning requiring external data | Tool calling |
| Static extraction or classification from user text | `@Generable` |
| Dynamic tool schemas defined at runtime | Dynamic tool + `DynamicGenerationSchema` |
