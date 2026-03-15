# Architecture

## The Three-Layer Rule

| Layer | Responsibility | May import FoundationModels? |
|---|---|---|
| Infrastructure | `LanguageModelSession`, network, disk | YES — exclusively here |
| Use Case | Coordinates domain + infrastructure | NO — depends on protocol |
| Domain | Pure business logic | NO — zero AI dependency |

**Violation — session instantiated in use case:**
```swift
// BAD: use case imports and instantiates LanguageModelSession directly
final class SummaryInteractor {
    private let session = LanguageModelSession(instructions: "Summarize text.")

    func summarize(_ text: String) async throws -> Summary { ... }
}
```

**Fix — express infrastructure as a protocol:**
```swift
// Infrastructure abstraction (use case layer owns the protocol)
protocol SummaryService {
    func summarize(_ text: String) async throws -> Summary
}

// Use case depends on protocol only
final class SummaryInteractor {
    private let service: SummaryService
    init(service: SummaryService) { self.service = service }

    func summarize(_ text: String) async throws -> Summary {
        try await service.summarize(text)
    }
}

// Concrete implementation (infrastructure layer) — only file that imports FoundationModels
import FoundationModels

final class FoundationModelsSummaryService: SummaryService {
    private let session = LanguageModelSession(instructions: "Summarize the provided text concisely.")

    func summarize(_ text: String) async throws -> Summary {
        try await session.respond(to: text, generating: Summary.self).content
    }
}
```

This boundary is what makes deterministic testing possible — see `references/testing.md` for the stub pattern.

## Functional Core, Imperative Shell

LLM calls are impure side effects — they belong at the infrastructure/use-case boundary, not inside domain logic.

```swift
// GOOD: domain model is pure — no async, no FoundationModels import
struct SummaryValidator {
    func isAcceptableLength(_ summary: Summary) -> Bool {
        summary.text.split(separator: " ").count <= 50
    }
}

// Use case fetches from model, passes result into pure validator
func summarizeAndValidate(_ text: String) async throws -> ValidationResult {
    let summary = try await service.summarize(text)       // impure — at the shell
    return validator.isAcceptableLength(summary)           // pure — domain core
}
```

## Command-Query Separation

AI calls that return data are **queries** — they must not mutate state.

```swift
// BAD: query mutates session history as a side effect AND deletes cache
func fetchSummary() async throws -> Summary {
    let result = try await session.respond(to: prompt, generating: Summary.self)
    cache.invalidate()   // Side-effect in a query!
    return result.content
}

// GOOD: separate query from command
func fetchSummary() async throws -> Summary {
    try await service.summarize(currentText)
}

func invalidateSummaryCache() {
    cache.invalidate()
}
```

## Dependency Inversion at the Composition Root

Inject the concrete infrastructure type at the app entry point or scene setup — nowhere else.

```swift
// Composition root (e.g., App struct or Scene)
let service: SummaryService = FoundationModelsSummaryService()
let interactor = SummaryInteractor(service: service)
let viewModel = SummaryViewModel(interactor: interactor)
```

## Error Translation at the Boundary

The infrastructure adapter is also responsible for translating vendor-specific errors (`LanguageModelSession.GenerationError`) into domain errors that the use-case layer understands. This prevents `FoundationModels` types from leaking up the stack. See `references/error-handling.md` for the full pattern.
