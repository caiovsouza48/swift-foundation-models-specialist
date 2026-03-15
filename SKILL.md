---
name: swift-foundation-models-specialist
description: >
  Use this skill when working with Apple's Foundation Models framework on iOS 26 /
  macOS 26 — writing or reviewing Swift code that integrates on-device language models,
  designing structured generation with @Generable and @Guide, handling LLM-specific
  errors (guardrailViolation, exceededContextWindowSize, rateLimited), gating features
  on SystemLanguageModel availability, implementing the Tool protocol, or optimizing
  streaming and prewarming. Also use when the user asks to add Apple Intelligence to a
  Swift app, build an on-device AI feature for iPhone or Mac, or debug unexpected output
  from LanguageModelSession — even if they don't mention FoundationModels, @Generable,
  or WWDC25 by name. Do NOT use for CloudKit, Core ML image classifiers, Vision
  framework, NLP with CreateML, SiriKit, or any Swift code unrelated to on-device
  language model generation.
license: MIT
compatibility: Designed for Claude Code, Cursor, Codex, Gemini CLI, and any agent that supports the Agent Skills format. Requires iOS 26 / macOS 26 deployment target and Swift 6.2+.
metadata:
  author: caiodesouza
  version: "1.0"
---

# Swift Foundation Models

Review and write Swift code that integrates Apple's Foundation Models framework for correctness, architectural cleanliness, testability, and production safety. Report only genuine problems — do not nitpick or invent issues.

## Review Process

When reviewing existing code, load only the reference files relevant to the issues you encounter. Follow these steps in order:

1. Check architectural layering — see `references/architecture.md`.
2. Check LanguageModelSession setup and lifecycle — see `references/sessions.md`.
3. Validate structured output with @Generable and @Guide — see `references/structured-output.md`.
4. Check GenerationOptions, sampling strategy, and non-determinism control — see `references/generation-options.md`.
5. Validate all LLM-specific error paths are handled — see `references/error-handling.md`.
6. Ensure availability checking and graceful degradation — see `references/availability.md`.
7. Validate prompt and instruction design for correctness and security — see `references/prompt-design.md`.
8. Check testability (protocol abstractions, stubs, evaluation separation) — see `references/testing.md`.
9. Check Tool implementations for correctness and safety — see `references/tool-calling.md`.
10. Final performance and streaming check — see `references/performance.md`.

For a partial review, load only the reference files that apply to the step you are on.

## Writing New Code

When generating new Foundation Models integration code from scratch:

1. Start by reading `references/architecture.md` to set up the correct layering — protocol abstraction in the domain layer, LanguageModelSession adapter in the infrastructure layer.
2. Read `references/structured-output.md` if the feature requires structured responses. Prefer @Generable types over free-form text parsing in every case.
3. Read `references/availability.md` to scaffold the availability check and fallback UI before writing any generation logic.
4. Produce one Swift file per type. Place infrastructure adapters, domain protocols, and view models in separate folders following a feature-based structure. This keeps Foundation Models integration modular and replaceable without touching the rest of the app.
5. Include LLM-specific error handling from the start — see `references/error-handling.md`. Bolting it on later is easy to forget and leads to silent failures in production.

If the user asks for a single quick snippet rather than a full feature, it is fine to produce a minimal self-contained example, but still call out where the session and @Generable types would live in a real project.

## Core Rules

**iOS 26 / macOS 26 is the default deployment target.** The Foundation Models framework requires Apple Intelligence-capable hardware.

**Target Swift 6.2+ with strict concurrency.** All LanguageModelSession calls are async and must be awaited correctly.

**Isolate LanguageModelSession to the infrastructure layer.** LanguageModelSession and all FoundationModels types belong exclusively in infrastructure adapters. Never let them leak into domain models, use-case coordinators, or view models — doing so couples your domain logic to Apple's framework and makes unit testing impossible without a physical Apple Intelligence device.

**Depend on protocols, not on LanguageModelSession directly.** Every AI integration the use-case layer touches should be expressed behind a protocol. This enables dependency injection, makes the feature testable with simple stubs, and keeps the door open if the backing implementation ever changes (e.g., a cloud fallback).

**Use @Generable and @Guide for all structured output.** Never parse free-form LLM text in production code. The model's phrasing is non-deterministic and will break string-based parsing across OS updates, locale changes, or minor prompt tweaks. Constrained decoding with @Generable guarantees the shape of the output at the type level.

**Prefer on-device capabilities.** Do not introduce third-party AI frameworks or cloud LLM SDKs without asking the user first. The Foundation Models framework is designed for privacy-preserving, low-latency, on-device generation.

**One type per file, feature-based folders.** When adding Foundation Models integration, place infrastructure adapters, domain protocols/models, and presentation-layer view models in separate files and folders. This keeps the boundary between layers visible and enforced by file structure rather than convention alone.

## Output Format

Organize findings by file. For each issue:

1. State the file and relevant line(s).
2. Name the rule being violated (e.g., "Isolate LanguageModelSession to the infrastructure layer").
3. Show a brief before/after code fix.

Skip files with no issues. End with a prioritized summary of the most impactful changes.

### Example

#### HealthCoachViewModel.swift

**Line 8: LanguageModelSession must not be instantiated in the presentation layer.**

```swift
// Before
@Observable
final class HealthCoachViewModel {
    private let session = LanguageModelSession(
        instructions: "You are a health coach."
    )

    func generateTip() async throws -> String {
        let response = try await session.respond(to: "Give me a tip")
        return response.content
    }
}

// After — protocol in domain layer; session moves to infrastructure
protocol HealthCoachService {
    func generateTip(for goal: HealthGoal) async throws -> HealthTip
}

@Observable
final class HealthCoachViewModel {
    private let service: HealthCoachService

    init(service: HealthCoachService) {
        self.service = service
    }

    func generateTip() async throws {
        tip = try await service.generateTip(for: currentGoal)
    }
}
```

---

**Line 22: Free-form text parsing is fragile — use @Generable for structured output.**

```swift
// Before
let response = try await session.respond(to: prompt)
let lines = response.content.split(separator: "\n")
let tip = lines.first ?? ""

// After
@Generable
struct HealthTip {
    @Guide(description: "One actionable tip in a single sentence")
    var actionableStep: String

    @Guide(description: "Estimated minutes per day", .range(1...60))
    var dailyMinutes: Int
}

let response = try await session.respond(to: prompt, generating: HealthTip.self)
```

---

**Line 31: Missing LLM-specific error handling.** Unhandled `guardrailViolation` or `exceededContextWindowSize` errors produce silent failures. Wrap generation calls with specific catch clauses:

```swift
// After
do {
    let tip = try await service.generateTip(for: goal)
    self.currentTip = tip
} catch LanguageModelSession.GenerationError.guardrailViolation {
    self.state = .contentUnavailable
} catch LanguageModelSession.GenerationError.exceededContextWindowSize {
    self.state = .sessionExpired
} catch {
    self.state = .failed(error)
}
```

---

#### Summary

1. **Architecture (critical):** LanguageModelSession in ViewModel breaks the infrastructure boundary and makes unit testing impossible.
2. **Structured output (high):** Free-form text parsing will break on any phrasing change; replace with @Generable.
3. **Error handling (high):** Missing LLM-specific error paths will produce silent failures in production.

## References

All reference files live in `references/`. Each review step above names the file to load. For code generation, start with `architecture.md`, then load files relevant to the feature you are building. For a partial review, load only the files that apply to the step you are on.

- `references/architecture.md` — Layering rules, infrastructure boundary, protocol-based abstraction, Functional Core / Imperative Shell, dependency inversion.
- `references/sessions.md` — LanguageModelSession lifecycle, init parameters (instructions, model, tools, transcript, guardrails), multi-turn conversations, transcript management, prewarming, model adapters.
- `references/structured-output.md` — @Generable macro, @Guide constraints (description, .count, .maximumCount, .range, .anyOf, Regex), constrained decoding guarantees, property ordering, DynamicGenerationSchema.
- `references/generation-options.md` — GenerationOptions, .greedy sampling for determinism, .random for reproducible variance, temperature tuning.
- `references/error-handling.md` — All 7 GenerationError cases (guardrailViolation, exceededContextWindowSize, unsupportedLanguageOrLocale, assetsUnavailable, unsupportedGuide, decodingFailure, rateLimited), ToolCallError, context window recovery tiers, .permissiveContentTransform guardrails.
- `references/availability.md` — SystemLanguageModel.default.availability, fallback UI patterns, device capability gating.
- `references/prompt-design.md` — Instructions vs. prompts, security (never embed user input in instructions), Swiss Cheese safety model, #Playground macro, output length control, few-shot examples.
- `references/testing.md` — Protocol stubs for unit tests, greedy sampling for integration snapshots, evaluation pipelines, seed-based reproducibility.
- `references/tool-calling.md` — Tool protocol, call(arguments:) and call(arguments:context:) variants, @Generable arguments, ToolOutput, stateful tools, ToolCallError, parallel invocations, privacy.
- `references/performance.md` — Streaming with streamResponse, snapshot vs. delta streaming, SwiftUI partial rendering, Instruments profiling.
