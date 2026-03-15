# Availability

## Checking Availability

```swift
switch SystemLanguageModel.default.availability {
case .available:
    // Show AI-powered UI
case .unavailable(let reason):
    handleUnavailable(reason: reason)
}
```

## Unavailability Reasons

| Reason | Meaning | Recommended Response |
|---|---|---|
| `.deviceNotEligible` | Hardware does not support Apple Intelligence | Hide the feature permanently |
| `.appleIntelligenceNotEnabled` | User has not enabled Apple Intelligence in Settings | Show settings deep-link prompt |
| `.modelNotReady` | Model is downloaded but still being prepared | Show loading state, retry after delay |

```swift
func handleUnavailable(reason: SystemLanguageModel.Availability.UnavailableReason) {
    switch reason {
    case .deviceNotEligible:
        featureState = .permanentlyUnavailable
    case .appleIntelligenceNotEnabled:
        featureState = .requiresSetup(settingsURL: URL(string: UIApplication.openSettingsURLString)!)
    case .modelNotReady:
        featureState = .loading
        scheduleRetry(after: .seconds(30))
    @unknown default:
        featureState = .unavailable
    }
}
```

## Design Fallbacks for Every AI Feature

```swift
var body: some View {
    switch featureState {
    case .available(let tip):
        AISuggestedTipView(tip: tip)
    case .loading:
        ProgressView("Preparing AI features…")
    case .requiresSetup(let url):
        EnableAIPromptView(settingsURL: url)
    case .permanentlyUnavailable, .unavailable:
        StaticTipView(tip: StaticTipProvider.defaultTip())
    }
}
```

## Check Availability Once, Cache the Result

Availability does not change during an app session (except `modelNotReady` → `available` transitions). Check once at feature initialisation, not before every inference call. The availability check is a synchronous property read and does not trigger model loading, so there is no latency concern.

```swift
@Observable
final class FeatureGate {
    private(set) var isAIAvailable = false

    func checkAvailability() {
        if case .available = SystemLanguageModel.default.availability {
            isAIAvailable = true
        } else {
            isAIAvailable = false
        }
    }
}
```

## Language Support Check

Check supported languages before allowing users to interact with a language-sensitive AI feature.

```swift
let supported = SystemLanguageModel.default.supportedLanguages
let currentLanguage = Locale.current.language

if supported.contains(where: { $0.languageCode == currentLanguage.languageCode }) {
    enableAIFeature()
} else {
    showLanguageNotSupportedMessage()
}
```
