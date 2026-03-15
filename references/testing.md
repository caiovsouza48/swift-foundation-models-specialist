# Testing

## The Core Principle

Acceptance tests for Foundation Models features test **your code's behavior**, not the model's output quality. These are different questions answered by different tools.

| Question | Tool | Runs on every PR? |
|---|---|---|
| Does my use case coordinate correctly? | Unit test with stub | Yes |
| Does my feature handle model errors? | Unit test with stub | Yes |
| Does the model return valid structure? | Integration test, greedy sampling | Weekly |
| Is the model's output good? | Evaluation pipeline | Weekly |

## The Test Pyramid for AI Features

```
        [ Evaluation Pipeline ]          ← quality scoring, real model, weekly
              (prompt quality, tone, accuracy)
         _________________________________
        [ Integration Tests ]             ← structural contracts, real model, weekly
          (greedy sampling, @Generable shape)
     _______________________________________________
    [ Acceptance Tests with Protocol Stubs ]        ← feature behavior, no model, every PR
      (use-case coordination, error paths, UI states)
 ___________________________________________________________
[ Unit Tests — Domain + Use Case Logic ]                     ← pure logic, every PR
  (no FoundationModels import, fully deterministic)
```

## Protocol Stubs for Deterministic Testing

The protocol boundary from `references/architecture.md` is what makes deterministic testing possible. Stubs return fixtures — no model required, no Apple Intelligence hardware needed, fully deterministic.

```swift
final class StubHealthCoachService: HealthCoachService {
    var result: Result<HealthTip, Error>

    init(returning tip: HealthTip) {
        self.result = .success(tip)
    }

    init(throwing error: Error) {
        self.result = .failure(error)
    }

    func generateTip(for goal: HealthGoal) async throws -> HealthTip {
        try result.get()
    }
}

extension HealthTip {
    static let fixture = HealthTip(
        actionableStep: "Drink a glass of water before each meal.",
        dailyMinutes: 5,
        difficulty: .easy
    )
}
```

## Acceptance Test Pattern (Red-Green-Refactor)

```swift
// RED: write the test first — HealthCoachInteractor doesn't exist yet
func test_generateTip_forHydrationGoal_returnsTip() async throws {
    let stub = StubHealthCoachService(returning: .fixture)
    let sut = HealthCoachInteractor(service: stub)

    let result = try await sut.loadDailyTip(for: .hydration)

    XCTAssertFalse(result.actionableStep.isEmpty)
    XCTAssertGreaterThan(result.dailyMinutes, 0)
}

// GREEN: implement the minimum to pass
final class HealthCoachInteractor {
    private let service: HealthCoachService
    init(service: HealthCoachService) { self.service = service }

    func loadDailyTip(for goal: HealthGoal) async throws -> HealthTip {
        try await service.generateTip(for: goal)
    }
}
```

## Test All LLM-Specific Error Paths

These are deterministic with stubs — they don't require the real model. Test every domain error that maps to a `GenerationError` case (see `references/error-handling.md` for the full list of 7+ error cases).

```swift
func test_whenGuardrailViolated_interactorThrowsContentUnavailable() async throws {
    let stub = StubHealthCoachService(throwing: HealthCoachError.contentUnavailable)
    let sut = HealthCoachInteractor(service: stub)

    do {
        _ = try await sut.loadDailyTip(for: .hydration)
        XCTFail("Expected error")
    } catch HealthCoachError.contentUnavailable {
        // Pass
    }
}

func test_whenModelUnavailable_viewModelShowsFallbackState() async throws {
    let stub = StubHealthCoachService(throwing: HealthCoachError.deviceNotSupported)
    let sut = HealthCoachViewModel(interactor: HealthCoachInteractor(service: stub))

    await sut.loadTip()

    XCTAssertEqual(sut.state, .fallback)
}

func test_whenRateLimited_viewModelShowsRetryState() async throws {
    let stub = StubHealthCoachService(throwing: HealthCoachError.rateLimited)
    let sut = HealthCoachViewModel(interactor: HealthCoachInteractor(service: stub))

    await sut.loadTip()

    XCTAssertEqual(sut.state, .retrying)
}

func test_whenAssetsUnavailable_viewModelShowsDownloadingState() async throws {
    let stub = StubHealthCoachService(throwing: HealthCoachError.modelNotReady)
    let sut = HealthCoachViewModel(interactor: HealthCoachInteractor(service: stub))

    await sut.loadTip()

    XCTAssertEqual(sut.state, .modelDownloading)
}
```

## Integration Tests — Structural Contracts

When testing against the real model, assert structure only — never assert on exact prose. Use `.greedy` sampling for repeatability across runs on the same OS version.

```swift
func test_realModel_generatesTipWithValidStructure() async throws {
    let service = FoundationModelsHealthCoachService()
    let tip = try await service.generateTip(for: .hydration)

    // Assert structure, NOT content
    XCTAssertFalse(tip.actionableStep.isEmpty)
    XCTAssert((1...60).contains(tip.dailyMinutes))
    XCTAssert([.easy, .moderate, .hard].contains(tip.difficulty))
}
```

Mark integration tests so they don't run on every PR:

```swift
// swift-testing tag
@Test(.tags(.integration))
func generatesTipWithValidStructure() async throws { ... }
```

**Important:** `.greedy` is deterministic within the same OS version, but an Apple model update can change the output. When greedy integration tests break after an OS update, that's a model drift signal — tune your prompts, don't treat it as a code regression.

## Evaluation Pipelines — Quality, Not Correctness

Prompt quality evaluation is NOT a unit test. It runs in a separate CI job on a schedule (e.g., weekly). Its purpose is to detect quality regressions after OS updates or prompt changes.

```swift
struct HealthTipEvaluator {
    let dataset: [(goal: HealthGoal, criteria: QualityCriteria)]
    let service: HealthCoachService

    func run() async throws -> EvaluationReport {
        var results: [EvaluationResult] = []
        for (goal, criteria) in dataset {
            let tip = try await service.generateTip(for: goal)
            let score = criteria.evaluate(tip)
            results.append(EvaluationResult(goal: goal, score: score))
        }
        return EvaluationReport(results: results)
    }
}
```

For large datasets, use an LLM-as-judge pattern (a second model grades each response). For small datasets, manual review is sufficient. Track scores over time — a drop after an OS update means prompt tuning is needed.

Use `#Playground` in Xcode (see `references/prompt-design.md`) for rapid prompt iteration during development, and the evaluation pipeline for ongoing quality monitoring in CI.

## What NOT to Test

- **Exact model output text** — it will change with OS updates; use structural assertions instead
- **Whether the model "understands" a concept** — out of scope for unit tests
- **Grammar or phrasing of model responses** — evaluation concern, not test concern
- **Greedy output stability across OS versions** — it's expected to drift; that's what evaluation pipelines catch
