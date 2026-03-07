# Loom Voice iOS — Claude Code Project Notes

## What This Is

Native iOS voice assistant client forked from VoxClaw. Connects to an existing Python voice server (`always_on_voice.py`) via LiveKit for bidirectional voice communication. The server handles all the intelligence (STT → Claude → TTS). This app is just the native iOS client: mic in, speaker out, stay alive in background.

**Bundle ID:** `com.ericday.loomvoice`
**Platform:** iOS 17+
**Package Manager:** Swift Package Manager (no Xcode project file)

## What We Forked From

This was VoxClaw — a macOS text-to-speech teleprompter app. We're keeping:
- Project structure (SPM)
- `PeerBrowser.swift` (Bonjour discovery — adapting service type)
- `KeychainHelper.swift` (secure storage)
- `SettingsManager.swift` (settings persistence pattern)
- `Log.swift` (logging)
- iOS app target (`VoxClawIOS/`)

We're removing all TTS, teleprompter, HTTP listener, overlay, CLI, and macOS code.

## Architecture

```
iOS App (LiveKit client)
  ├── RoomManager (@Observable) — owns LiveKit Room, publishes state
  ├── ServerAPI — HTTP client for /token, /health, /model, /tts, /interrupt
  ├── AudioSessionManager — AVAudioSession config + interruption handling
  ├── ConnectionManager — reconnection state machine + NWPathMonitor
  ├── NotificationManager — local notifications on disconnect
  └── Views (SwiftUI)
      ├── VoiceView — main screen with animated orb
      ├── OrbView — Canvas-based animated orb (state-driven, audio-reactive)
      └── SettingsView — server URL, audio prefs, connection status

Server (DO NOT MODIFY — already exists)
  ├── LiveKit room "loom-voice" on wss://....:7880
  ├── HTTP on https://....:8089
  │   ├── GET /token?deviceId=<uuid>
  │   ├── GET /health
  │   ├── POST /model
  │   ├── POST /tts
  │   └── POST /interrupt
  └── STT (Whisper) → LLM (Claude) → TTS (Kokoro)
```

## Build & Test

```bash
swift build                              # Debug build (iOS simulator)
swift test                               # Run remaining tests
xcodebuild -scheme VoxClawIOS -sdk iphonesimulator build   # Full iOS build
```

Note: The Xcode scheme name may still say VoxClawIOS until Story 10 (rebrand).

## Key Dependencies

- **LiveKit Swift SDK** (`client-sdk-swift` >= 2.12.1) — WebRTC room, audio tracks
- No other external dependencies for MVP

## Server Endpoints (Reference — DO NOT MODIFY SERVER)

```bash
# Get LiveKit token
curl https://erics-macbook-pro.tail893ab1.ts.net:8089/token?deviceId=<uuid>
# Response: { "token": "...", "url": "wss://...", "room": "loom-voice", "version": "0.4.4" }

# Health check
curl https://erics-macbook-pro.tail893ab1.ts.net:8089/health
# Response: { "status": "ok", "room": "loom-voice", "participants": [...] }

# Switch LLM model
curl -X POST .../model -d '{"model": "claude-haiku"}'

# Switch TTS engine
curl -X POST .../tts -d '{"engine": "kokoro", "voice": "bella"}'

# Interrupt current TTS
curl -X POST .../interrupt
```

## Key Conventions

- **@Observable** on state classes (RoomManager, etc.) — use stored properties with didSet, not computed
- **@MainActor** on anything that drives UI
- **Structured concurrency** — use Task, async/await, proper cancellation
- **SF Symbols** for all icons — no custom image assets
- **Dark mode only** for MVP
- **iOS 17+ minimum** — can use latest SwiftUI features
- Tests use Swift Testing framework (`@Test`, `#expect`) where applicable
- Logging via `os.Logger` categories in `Log.swift`

## Important iOS Notes

- `UIBackgroundModes: audio, voip` in Info.plist for background persistence
- Let LiveKit SDK manage AVAudioSession (`isAutomaticConfigurationEnabled = true`)
- Handle `AVAudioSession.interruptionNotification` for phone calls / Siri
- Use `NWPathMonitor` for network transition detection (WiFi ↔ cell)
- Token endpoint is unauthenticated — Tailscale network IS the auth boundary

## Codebase Patterns

(Ralph will populate this as patterns are discovered during implementation)
