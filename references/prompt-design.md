# Prompt Design

## Instructions vs Prompts — A Hard Boundary

| | Instructions | Prompts |
|---|---|---|
| **Author** | Developer only | Developer + user input |
| **Persistence** | All turns in the session | Single turn |
| **Scope** | Persona, behavior, rules | Task input |
| **Precedence** | Take precedence over prompts | Subordinate to instructions |
| **User input allowed?** | NEVER | Yes |

> Source: WWDC25 Session 286 — "Meet the Foundation Models framework": "Instructions should come from you, the developer, while prompts can come from the user. The model is trained to obey instructions over prompts. This helps protect against prompt injection attacks, but is by no means bulletproof."

```swift
// GOOD: instructions are developer-owned, prompt carries user input
let session = LanguageModelSession(
    instructions: """
    You are a recipe assistant.
    Respond only with ingredient lists in a numbered format.
    Do not include cooking instructions.
    """
)
let response = try await session.respond(to: userQuery)  // User text goes in prompt

// BAD: user input inside instructions — prompt injection risk
let session = LanguageModelSession(
    instructions: "You are a recipe assistant. The user asked: \(userQuery)"  // NEVER
)
```

## Handling User Input Safely — Three Patterns

WWDC25 Session 248 — "Explore prompt design & safety for on-device foundation models" describes three approaches for handling user input, ordered from lowest to highest risk. The labels below (curated, template, direct) are shorthand — Session 248 describes these as levels of flexibility, not as named categories.

**Pattern 1 — Curated prompts (lowest risk)**
The user selects from a fixed list. No free-form input reaches the model.

```swift
let builtInPrompts = ["Fox story", "Cat adventure", "Bird journey"]
let selectedPrompt = builtInPrompts[selectedIndex]
let response = try await session.respond(to: selectedPrompt)
```

**Pattern 2 — Template with user input (medium risk)**
User input fills a controlled slot inside a developer-defined template.

```swift
let prompt = "Generate a bedtime story featuring: \(userInput)"
let response = try await session.respond(to: prompt)
```

**Pattern 3 — Direct user input (highest risk)**
User text goes directly to the model as the prompt. Flexible, but requires robust safety instructions and guardrail awareness.

```swift
let response = try await session.respond(to: userInput)
```

Choose the lowest-risk pattern that still delivers the feature. Combine patterns where possible — for example, a template prompt (Pattern 2) with a curated instruction set.

## Writing Effective Instructions

### Be Specific About Output Format

```swift
// Vague
instructions: "Help with fitness."

// Specific
instructions: """
You are a fitness coach. For each response:
- Provide exactly one exercise recommendation
- Include the target muscle group
- Keep your response under 50 words
"""
```

### Use Positive and Negative Directives

> Source: WWDC25 Session 248 recommends writing "DO NOT" in ALL CAPS — the model responds well to emphatic negation.

```swift
instructions: """
You are a children's story assistant.
DO write stories suitable for ages 6–10.
DO use simple vocabulary.
DO NOT include violence, fear, or adult themes.
DO NOT make medical claims.
"""
```

### Control Output Length with Explicit Phrases

```swift
// Shorter output
"Generate a summary in three sentences."
"Describe this in a few words."

// Longer output
"Explain this in detail."
```

### Provide Concrete Examples — Fewer Is Better

> Source: WWDC25 Session 248 recommends fewer than 5 examples. More examples increase token cost without proportional quality gain.

```swift
instructions: """
You extract action items from meeting notes.
Format each as: [Owner] — [Action] — by [Date].

Example:
Input: "Alice will update the design by Friday"
Output: Alice — Update the design — by Friday
"""
```

## Safety Layers — Swiss Cheese Model

> Source: WWDC25 Session 248 uses the Swiss Cheese analogy: safety is multi-layered. Each layer has gaps, but misaligned gaps across layers prevent failures from passing through.

**Layer 1 — Framework guardrails (Apple-provided)**
Applied automatically to both inputs and outputs. Cannot be disabled. First line of defense.

**Layer 2 — Developer instructions**
Add safety guidance directly in instructions. Never include user input here.

```swift
instructions: """
You are a wellness assistant.
Respond to sensitive topics in an empathetic and supportive way.
DO NOT provide medical diagnoses or treatment recommendations.
"""
```

**Layer 3 — User input design**
Control how user input reaches the model. Use curated or template patterns (see above) to reduce attack surface.

**Layer 4 — Use-case-specific mitigations**
Anticipate the real-world consequences of your feature's output and build specific safeguards.

```swift
// Example: recipe app generating content that may include allergens
instructions: """
You are a recipe assistant.
Always note the top 8 allergens (nuts, dairy, gluten, eggs, soy, fish, shellfish, sesame)
that appear in each recipe.
"""
```

> Session 248 checklist: handle guardrail errors, include safety in instructions, balance flexibility vs. safety in input handling, anticipate real-world impact, implement use-case-specific mitigations, evaluate and test thoroughly, report issues via Feedback Assistant.

## Prompt Injection Defense

Never concatenate user input into instructions. When user preferences must influence behavior, validate and sanitize the input, then pass it in the prompt — not in instructions.

```swift
// BAD: user controls the model's behavior via instructions
func buildSession(userPreferences: String) -> LanguageModelSession {
    LanguageModelSession(instructions: "Be a \(userPreferences) assistant.")
}

// GOOD: user preference is data in the prompt
let session = LanguageModelSession(
    instructions: "You are a cooking assistant. Adapt suggestions to the dietary preference the user specifies in each message."
)
let response = try await session.respond(
    to: "My dietary preference: \(sanitized(userPreferences)). Suggest a dinner."
)
```

## On-Device Model Scope

The on-device model is optimized for:
- Summarization
- Classification and tagging
- Entity extraction
- Short-form creative text
- Guided structured generation

It is NOT designed for:
- World knowledge or recent facts (use tool calling instead)
- Complex multi-step reasoning
- Long-form content generation
- Mathematical computation
- Code generation

Prompt for what the model is good at. Use Tool Calling (see `references/tool-calling.md`) to fill gaps with real data from your app or system APIs.

## Prompt Verbosity and Token Cost

Every token in instructions adds latency. Keep them concise.

```swift
// Verbose
instructions: "You are a very helpful and friendly assistant that helps users with all of their questions about their health and fitness goals and you should always be positive and encouraging..."

// Concise
instructions: "You are a positive, encouraging health and fitness assistant."
```
