# CLAUDE.md - VoiceInk AI Assistant Guide

This document provides comprehensive guidance for AI assistants working with the VoiceInk codebase.

## Project Overview

VoiceInk is a **native macOS application** (Swift/SwiftUI) that provides voice-to-text transcription using local AI models. The app runs Whisper models locally for privacy-focused, offline transcription with optional AI enhancement through cloud providers.

**Key Value Propositions:**
- 100% offline transcription (privacy-first)
- AI enhancement via multiple providers (GROQ, OpenAI, Anthropic, Gemini, DeepSeek, Ollama)
- Power Mode: context-aware settings based on active app/URL
- Global keyboard shortcuts and push-to-talk

**Requirements:**
- macOS 14.0+
- Xcode (latest recommended)
- whisper.cpp XCFramework (see BUILDING.md)

## Codebase Structure

```
VoiceInk/
├── VoiceInk/                    # Main application source
│   ├── VoiceInk.swift           # App entry point (@main)
│   ├── AppDelegate.swift        # App lifecycle management
│   ├── Models/                  # Data models
│   │   ├── Transcription.swift  # SwiftData model for transcriptions
│   │   ├── PowerModeConfig.swift # Per-app configuration settings
│   │   ├── CustomPrompt.swift   # AI prompt definitions
│   │   └── PredefinedModels.swift # Whisper model definitions
│   ├── Services/                # Core business logic
│   │   ├── AIService.swift      # Multi-provider AI integration
│   │   ├── AIEnhancementService.swift # Text enhancement pipeline
│   │   ├── ActiveWindowService.swift  # App detection for Power Mode
│   │   ├── AudioDeviceManager.swift   # Audio input handling
│   │   ├── ScreenCaptureService.swift # Context-aware screen capture
│   │   ├── WordReplacementService.swift # Custom dictionary
│   │   └── BrowserURLService.swift    # Browser URL detection
│   ├── Whisper/                 # Whisper integration
│   │   ├── WhisperState.swift   # Main transcription state machine
│   │   ├── LibWhisper.swift     # whisper.cpp bindings
│   │   └── WhisperPrompt.swift  # Prompt configuration
│   ├── Views/                   # SwiftUI views
│   │   ├── ContentView.swift    # Main window
│   │   ├── MiniRecorderView.swift    # Floating recorder UI
│   │   ├── NotchRecorderView.swift   # MacBook notch recorder
│   │   ├── Settings/            # Settings panels
│   │   ├── Onboarding/          # First-run experience
│   │   └── Components/          # Reusable UI components
│   ├── ViewModels/              # View state management
│   │   └── LicenseViewModel.swift
│   ├── Resources/               # Assets and sounds
│   │   └── Sounds/
│   └── *.swift                  # Core managers (Hotkey, Clipboard, etc.)
├── VoiceInkTests/               # Unit tests
├── VoiceInkUITests/             # UI tests
├── VoiceInk.xcodeproj/          # Xcode project
└── *.md                         # Documentation
```

## Key Architecture Components

### 1. App Initialization (VoiceInk.swift)

The app uses `@main` entry point with dependency injection:
```swift
@main
struct VoiceInkApp: App {
    @NSApplicationDelegateAdaptor(AppDelegate.self) var appDelegate
    let container: ModelContainer  // SwiftData
    @StateObject private var whisperState: WhisperState
    @StateObject private var aiService: AIService
    @StateObject private var enhancementService: AIEnhancementService
    // ... other services
}
```

Services are initialized in specific order due to dependencies:
1. `AIService` (standalone)
2. `AIEnhancementService` (depends on AIService, ModelContext)
3. `WhisperState` (depends on ModelContext, AIEnhancementService)
4. `HotkeyManager` (depends on WhisperState)
5. `MenuBarManager` (depends on all above)

### 2. Transcription Pipeline (WhisperState.swift)

The core transcription flow:
1. User triggers recording (hotkey/push-to-talk)
2. `Recorder` captures audio to WAV file
3. Whisper model loads (lazy loading, unloads after use)
4. `transcribeAudio()` processes the recording
5. Optional: `AIEnhancementService.enhance()` improves text
6. Text is pasted at cursor via `CursorPaster`
7. Transcription saved to SwiftData

**Key state properties:**
- `isRecording`: Currently capturing audio
- `isTranscribing`: Processing with Whisper
- `isProcessing`: Any processing happening
- `isMiniRecorderVisible`: Recorder UI visible

### 3. AI Provider System (AIService.swift)

Supports multiple providers via `AIProvider` enum:
```swift
enum AIProvider: String, CaseIterable {
    case groq, openAI, deepSeek, gemini, anthropic, ollama, custom
}
```

Each provider has:
- `baseURL`: API endpoint
- `defaultModel`: Default model name
- `requiresAPIKey`: Whether API key needed (Ollama = false)

API keys stored in UserDefaults with pattern: `"{Provider}APIKey"`

### 4. Power Mode (PowerModeConfig.swift)

Automatically applies settings based on active application:
```swift
struct PowerModeConfig {
    let bundleIdentifier: String
    var isAIEnhancementEnabled: Bool
    var selectedPrompt: String?  // UUID
    var urlConfigs: [URLConfig]? // URL-specific overrides
}
```

`ActiveWindowService` detects frontmost app and applies config.

### 5. Data Persistence

**SwiftData** for transcription history:
```swift
@Model
final class Transcription {
    var id: UUID
    var text: String
    var enhancedText: String?
    var timestamp: Date
    var duration: TimeInterval
    var audioFileURL: String?
}
```

**UserDefaults** for settings (pattern: camelCase keys)

**File storage locations:**
- Models: `~/Library/Application Support/com.prakashjoshipax.VoiceInk/WhisperModels/`
- Recordings: `~/Library/Application Support/com.prakashjoshipax.VoiceInk/Recordings/`
- SwiftData: `~/Library/Application Support/com.prakashjoshipax.VoiceInk/default.store`

## Development Workflow

### Building

1. Clone whisper.cpp and build XCFramework:
   ```bash
   git clone https://github.com/ggerganov/whisper.cpp.git
   cd whisper.cpp && ./build-xcframework.sh
   ```

2. Add `whisper.xcframework` to Xcode project

3. Build with Cmd+B, Run with Cmd+R

### Testing

Run tests via Xcode (Cmd+U) or:
```bash
xcodebuild test -scheme VoiceInk -destination 'platform=macOS'
```

The test suite uses Swift Testing framework (`import Testing`).

### Debugging Tips

- Enable Console.app logging with filter: `com.prakashjoshipax.voiceink`
- Check UserDefaults: `defaults read com.prakashjoshipax.VoiceInk`
- Model loading issues: Check `~/Library/Application Support/com.prakashjoshipax.VoiceInk/WhisperModels/`

## Code Conventions

### Swift Style

- **Actor isolation**: Main classes use `@MainActor` (WhisperState, HotkeyManager)
- **Async/await**: Preferred for async operations
- **Published properties**: Use `@Published` for observable state
- **Dependency injection**: Pass services through initializers
- **Singletons**: Use sparingly (`PowerModeManager.shared`, `WordReplacementService.shared`)

### Naming Conventions

- **UserDefaults keys**: camelCase (e.g., `isAIEnhancementEnabled`, `selectedAIProvider`)
- **Notification names**: camelCase with descriptive prefix (e.g., `.toggleMiniRecorder`, `.aiProviderKeyChanged`)
- **Services**: Suffix with `Service` (e.g., `AIEnhancementService`)
- **Managers**: Suffix with `Manager` for coordinating classes

### Error Handling

Custom error enums per domain:
```swift
enum EnhancementError: Error {
    case notConfigured, emptyText, invalidResponse, ...
}

enum WhisperStateError: Error {
    case modelLoadFailed, transcriptionFailed, ...
}
```

### Logging

Use `os.Logger` for structured logging:
```swift
private let logger = Logger(
    subsystem: "com.prakashjoshipax.VoiceInk",
    category: "categoryName"
)
logger.notice("Message with \(value, privacy: .public)")
```

## Key Dependencies

| Dependency | Purpose | Integration |
|------------|---------|-------------|
| whisper.cpp | Local transcription | XCFramework |
| Sparkle | Auto-updates | SPM |
| KeyboardShortcuts | Hotkey management | SPM |
| LaunchAtLogin | Login item | SPM |

## Important Patterns

### State Machine Pattern (WhisperState)

Recording follows strict state transitions:
```
Idle -> Recording -> Transcribing -> (Enhancing) -> Idle
```

Always check state before transitions:
```swift
if shouldCancelRecording { return }
```

### Cleanup Pattern

Resources are cleaned after transcription:
```swift
private func cleanupResources() async {
    await whisperContext?.releaseResources()
    whisperContext = nil
    isModelLoaded = false
}
```

### Rate Limiting (AIEnhancementService)

Cloud API calls include rate limiting:
```swift
private let rateLimitInterval: TimeInterval = 1.0
private var lastRequestTime: Date?
```

### Retry with Exponential Backoff

Network requests retry with backoff:
```swift
let timeout = baseTimeout * pow(2.0, Double(retryCount))
```

## Permissions Required

The app requires these entitlements (VoiceInk.entitlements):
- Audio input (microphone)
- Screen capture (for context)
- Accessibility (for cursor pasting)
- Apple Events (for browser URL detection)
- Network client/server (for AI APIs)

## Common Tasks for AI Assistants

### Adding a New AI Provider

1. Add case to `AIProvider` enum in `AIService.swift`
2. Implement `baseURL` and `defaultModel`
3. Add verification method if needed (see `verifyGeminiAPIKey`)
4. Add handling in `AIEnhancementService.makeRequest()`
5. Update UI in `APIKeyManagementView.swift`

### Adding a New Setting

1. Add `@Published` property with UserDefaults backing:
   ```swift
   @Published var newSetting: Bool {
       didSet { UserDefaults.standard.set(newSetting, forKey: "newSetting") }
   }
   ```
2. Initialize from UserDefaults in `init()`
3. Add UI control in appropriate settings view

### Modifying Transcription Pipeline

Key touchpoints:
- `WhisperState.toggleRecord()` - Start/stop
- `WhisperState.transcribeAudio()` - Core pipeline
- `AIEnhancementService.enhance()` - Post-processing

### Adding Keyboard Shortcuts

1. Extend `KeyboardShortcuts.Name` in `HotkeyManager.swift`
2. Set shortcut with `KeyboardShortcuts.setShortcut()`
3. Add handler with `KeyboardShortcuts.onKeyDown()`

## Files to Avoid Modifying

- `LibWhisper.swift` - Auto-generated whisper.cpp bindings
- `*.entitlements` - Requires re-signing
- `project.pbxproj` - Managed by Xcode

## Testing Considerations

- Mock `AIService` for enhancement tests
- Use test audio files for transcription tests
- Test Power Mode with mock `NSRunningApplication`
- Verify UserDefaults cleanup between tests
