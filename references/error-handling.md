# Error Handling

## GenerationError — Complete Reference

`LanguageModelSession.GenerationError` is the primary error type. All cases carry an associated `Context` value with `debugDescription` and `underlyingErrors` properties.

| Error Case | Meaning |
|---|---|
| `.guardrailViolation(_:)` | The model refused because the prompt or context triggered a safety filter |
| `.exceededContextWindowSize(_:)` | Session transcript has grown too large (4096 token limit) |
| `.unsupportedLanguageOrLocale(_:)` | The device language or locale is not supported |
| `.assetsUnavailable(_:)` | Model assets are not downloaded or not ready |
| `.unsupportedGuide(_:)` | A `@Guide` constraint is invalid or unsupported by the current model |
| `.decodingFailure(_:)` | Structured output could not be decoded into the target `@Generable` type |
| `.rateLimited(_:)` | Too many requests were made in a short period |

> Note: The nested type `LanguageModelSession.GenerationError.Refusal` provides details for guardrail violations.

## Primary Error Categories

### 1. Guardrail Violation

```swift
do {
    tip = try await service.generateTip(for: goal)
} catch LanguageModelSession.GenerationError.guardrailViolation {
    state = .contentUnavailable("This content can't be generated on-device.")
}
```

**UX guidance:**
- For **proactive features** (auto-summaries, background suggestions): silently ignore the error and show nothing.
- For **user-initiated features** (explicit "generate" button): show a clear, non-alarming message.

### 2. Context Window Exceeded

The session's conversation history has grown too large. The on-device model has a fixed context window of 4096 tokens (input + output combined). The error case carries an associated `Context` value.

> Source: TN3193 — "Managing the on-device foundation model's context window" and WWDC25 Session 301 — "Deep dive into the Foundation Models framework" are the authoritative references.

Apple documents **three recovery tiers**. Use the one that matches your feature's context needs:

**Tier 1 — Fresh start (simple, no history preserved):**

Suitable for single-turn features or when prior context is irrelevant.

```swift
do {
    response = try await session.respond(to: prompt)
} catch LanguageModelSession.GenerationError.exceededContextWindowSize {
    session = LanguageModelSession(instructions: systemInstructions)
    response = try await session.respond(to: prompt)
}
```

**Tier 2 — Transcript carry-over (recommended for multi-turn features):**

Create a new session seeded with a condensed transcript: the first entry (instructions) and the last entry (most recent exchange). This preserves conversation continuity without replaying the full history.

```swift
do {
    response = try await session.respond(to: prompt)
} catch LanguageModelSession.GenerationError.exceededContextWindowSize {
    session = makeRecoverySession(from: session)
    response = try await session.respond(to: prompt)
}

private func makeRecoverySession(
    from old: LanguageModelSession
) -> LanguageModelSession {
    let allEntries = old.transcript.entries
    var condensedEntries = [Transcript.Entry]()

    if let firstEntry = allEntries.first {
        condensedEntries.append(firstEntry)              // Always keep instructions
        if allEntries.count > 1, let lastEntry = allEntries.last {
            condensedEntries.append(lastEntry)           // Keep most recent exchange
        }
    }

    let condensedTranscript = Transcript(entries: condensedEntries)
    return LanguageModelSession(transcript: condensedTranscript)
}
```

**Tier 3 — Proactive summarization (for long-running chat):**

At ~75% context capacity, summarize the conversation so far using a separate session, then start a fresh main session seeded with that summary. This avoids hitting the limit reactively.

```swift
private func summarizeHistory(_ transcript: Transcript) async throws -> String {
    let summarySession = LanguageModelSession(
        instructions: "Summarize the following conversation in 3 sentences."
    )
    let response = try await summarySession.respond(
        to: transcript.description,
        options: GenerationOptions(sampling: .greedy)
    )
    return response.content
}
```

### 3. Unsupported Language or Locale

```swift
do {
    response = try await session.respond(to: prompt)
} catch LanguageModelSession.GenerationError.unsupportedLanguageOrLocale {
    state = .languageNotSupported
}
```

Check `SystemLanguageModel.default.supportedLanguages` at feature initialisation to detect this condition before the first call.

### 4. Assets Unavailable

```swift
catch LanguageModelSession.GenerationError.assetsUnavailable {
    state = .modelNotReady
    // Prompt user to check Apple Intelligence settings or wait for download
}
```

### 5. Decoding Failure

```swift
catch LanguageModelSession.GenerationError.decodingFailure {
    state = .failed("Could not parse the model's response.")
    // Consider simplifying your @Generable type or @Guide constraints
}
```

### 6. Rate Limited

```swift
catch LanguageModelSession.GenerationError.rateLimited {
    state = .retrying
    try await Task.sleep(for: .seconds(2))
    // Retry the request
}
```

### 7. Unsupported Guide

```swift
catch LanguageModelSession.GenerationError.unsupportedGuide {
    // Fall back to a simpler @Generable type without the problematic constraint
}
```

## ToolCallError

Tool-specific failures are reported separately via `LanguageModelSession.ToolCallError`, which carries a reference to the failing `tool` and the `underlyingError`. Handle these when you need to distinguish tool failures from generation failures.

## Comprehensive Error Handler Pattern

```swift
func generate(prompt: String) async {
    do {
        let result = try await service.generate(prompt: prompt)
        state = .loaded(result)
    } catch LanguageModelSession.GenerationError.guardrailViolation {
        state = .unavailable
    } catch LanguageModelSession.GenerationError.exceededContextWindowSize {
        rotateSession()
        state = .retrying
    } catch LanguageModelSession.GenerationError.unsupportedLanguageOrLocale {
        state = .languageUnsupported
    } catch LanguageModelSession.GenerationError.assetsUnavailable {
        state = .modelNotReady
    } catch LanguageModelSession.GenerationError.decodingFailure {
        state = .failed("Could not parse the response.")
    } catch LanguageModelSession.GenerationError.rateLimited {
        state = .retrying
    } catch LanguageModelSession.GenerationError.unsupportedGuide {
        state = .failed("Unsupported model constraint.")
    } catch {
        state = .failed(error.localizedDescription)
    }
}
```

## Errors in the Infrastructure Layer

Catch and re-throw as domain errors at the infrastructure boundary. Use cases and domain models must never see `LanguageModelSession.GenerationError` directly — that would couple them to the infrastructure type, breaking the architectural boundary from `references/architecture.md`.

```swift
final class FoundationModelsHealthCoachService: HealthCoachService {
    func generateTip(for goal: HealthGoal) async throws -> HealthTip {
        do {
            let response = try await session.respond(
                to: goal.prompt,
                generating: HealthTip.self
            )
            return response.content
        } catch LanguageModelSession.GenerationError.guardrailViolation {
            throw HealthCoachError.contentUnavailable
        } catch LanguageModelSession.GenerationError.exceededContextWindowSize {
            session = LanguageModelSession(instructions: coachInstructions)
            throw HealthCoachError.sessionExpired
        } catch LanguageModelSession.GenerationError.unsupportedLanguageOrLocale {
            throw HealthCoachError.deviceNotSupported
        } catch LanguageModelSession.GenerationError.assetsUnavailable {
            throw HealthCoachError.modelNotReady
        } catch LanguageModelSession.GenerationError.rateLimited {
            throw HealthCoachError.rateLimited
        } catch {
            throw HealthCoachError.unknown(error)
        }
    }
}

enum HealthCoachError: Error {
    case contentUnavailable
    case sessionExpired
    case deviceNotSupported
    case modelNotReady
    case rateLimited
    case unknown(Error)
}
```

## Never Swallow Errors Silently

```swift
// BAD: error is eaten, user has no feedback
let response = try? await session.respond(to: prompt)
guard let content = response?.content else { return }

// GOOD: map to explicit state
do {
    let response = try await session.respond(to: prompt)
    state = .loaded(response.content)
} catch {
    state = .failed
}
```
