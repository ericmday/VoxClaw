# TASKS.md — Loom Voice iOS (Fork of VoxClaw)

## Project Overview

Convert VoxClaw (a one-directional text-to-speech teleprompter app) into **Loom Voice** — a bidirectional always-on voice assistant client. The server (`always_on_voice.py`) already handles STT → Claude → TTS. This app just needs to:

1. Connect to a LiveKit room
2. Stream mic audio to the server
3. Play back the server's audio response
4. Stay alive in background

**Full PRD:** `~/clawd/docs/loom-voice-ios-prd.md`

## What We're Keeping from VoxClaw
- Swift Package Manager project structure
- Bonjour/mDNS discovery (`PeerBrowser.swift`) — adapt service type to `_loom-voice._tcp`
- KeychainHelper for secure storage (repurpose for server secrets)
- Network layer patterns (NWListener → reuse for NWPathMonitor)
- Settings persistence approach (UserDefaults via SettingsManager)
- App icon assets and bundle structure
- iOS app target (`VoxClawIOS/`)

## What We're Removing
- All TTS playback code (OpenAI TTS, ElevenLabs TTS, Apple TTS services)
- Teleprompter/word-highlighting UI (ReadingSession, WordTimingEstimator, SpeechAligner)
- HTTP listener for receiving text (NetworkListener, NetworkSession — server push model gone)
- Overlay panel / floating window (macOS only, not needed)
- CLI parser (no CLI mode)
- The macOS app target entirely (iOS only for now)

## What We're Adding
- LiveKit Swift SDK (`client-sdk-swift` v2.12+)
- LiveKit room connection + audio track management
- AVAudioSession background audio handling
- Reconnection state machine with exponential backoff
- Animated orb UI (state-driven, audio-reactive)
- Token fetch from server HTTP endpoint
- NWPathMonitor for network transition handling

## Server Endpoints (already exist, don't modify)
- `GET /token?deviceId=<uuid>` → `{ token, url, room, version }`
- `GET /health` → `{ status, room, participants, uptime }`
- `POST /model` → `{ model: "claude-haiku" }` → switch LLM
- `POST /tts` → `{ engine: "kokoro", voice: "bella" }` → switch TTS
- `POST /interrupt` → interrupt current TTS playback
- Server: `https://erics-macbook-pro.tail893ab1.ts.net:8089`
- LiveKit: `wss://erics-macbook-pro.tail893ab1.ts.net:7880`

---

## Stories

### Story 1: Strip VoxClaw to Shell
**As a** developer  
**I want** to remove all VoxClaw-specific code (TTS services, teleprompter, HTTP listener, overlay, CLI, macOS target)  
**So that** I have a clean iOS app shell to build Loom Voice on top of

**Acceptance Criteria:**
- [ ] Remove `Sources/VoxClawCore/Audio/` (TTSService, ElevenLabsTTSService, FallbackSpeechEngine, AppleSpeechEngine, AudioPlayer)
- [ ] Remove `Sources/VoxClawCore/Reading/` (ReadingSession, WordTimingEstimator, SpeechAligner)
- [ ] Remove `Sources/VoxClawCore/Network/NetworkListener.swift` and `NetworkSession.swift` (HTTP server — we're a client now)
- [ ] Remove `Sources/VoxClawCore/Network/HTTPRequestParser.swift`
- [ ] Remove `Sources/VoxClawCore/Panel/` (FloatingPanel, PanelController — macOS overlay)
- [ ] Remove `Sources/VoxClawCore/Input/CLIParser.swift` and `ModeDetector.swift`
- [ ] Keep `PeerBrowser.swift` (Bonjour discovery — will adapt later)
- [ ] Keep `KeychainHelper.swift` (will repurpose)
- [ ] Keep `SettingsManager.swift` (will modify)
- [ ] Keep `AppState.swift` (will rewrite)
- [ ] Keep `Log.swift`
- [ ] Remove all macOS-only code paths, keep only iOS target (`VoxClawIOS/`)
- [ ] Update `Package.swift` to remove macOS platform target, keep iOS only
- [ ] App builds and launches (blank screen is fine)

---

### Story 2: Add LiveKit SDK + Room Connection
**As a** user  
**I want** the app to connect to a LiveKit room  
**So that** I can have a voice channel with the Loom server

**Acceptance Criteria:**
- [ ] Add `LiveKit` Swift package dependency (`client-sdk-swift` >= 2.12.1)
- [ ] Create `Managers/RoomManager.swift` — `@Observable` class, `@MainActor`
- [ ] RoomManager has `ConnectionState` enum: `.disconnected`, `.connecting`, `.connected`, `.reconnecting(attempt:)`, `.failed(Error)`
- [ ] Create `Managers/ServerAPI.swift` with `fetchToken(deviceId:)` method
- [ ] Device ID is a stable UUID stored in `@AppStorage("deviceId")`
- [ ] On launch: fetch token from `GET /token?deviceId=<uuid>`, then `room.connect(url:token:)`
- [ ] Configure `RoomOptions` with `AudioCaptureOptions(echoCancellation: true, noiseSuppression: true, autoGainControl: true)`
- [ ] Configure `AudioPublishOptions(dtx: true)` for bandwidth savings during silence
- [ ] Implement `RoomDelegate` for connection state changes
- [ ] Auto-publish microphone on connect: `room.localParticipant.setMicrophone(enabled: true)`
- [ ] Server URL defaults to `https://erics-macbook-pro.tail893ab1.ts.net:8089`
- [ ] LiveKit URL comes from token response (don't hardcode)
- [ ] Connection works over Tailscale

---

### Story 3: Audio Session + Background Persistence
**As a** user  
**I want** the voice connection to stay alive when I background the app  
**So that** Loom is always available without reopening the app

**Acceptance Criteria:**
- [ ] `Info.plist`: `UIBackgroundModes` includes `audio` and `voip`
- [ ] `Info.plist`: `NSMicrophoneUsageDescription` set
- [ ] Create `Managers/AudioSessionManager.swift`
- [ ] Let LiveKit SDK manage AVAudioSession automatically (`isAutomaticConfigurationEnabled = true`)
- [ ] Enable `setRecordingAlwaysPreparedMode(true)` for instant mic availability
- [ ] Set `duckingLevel = .min` to minimize ducking other audio
- [ ] Set mic mute mode to `.voiceProcessing` (fast mute, no audio engine restart)
- [ ] Handle `AVAudioSession.interruptionNotification` (began + ended)
- [ ] On interruption ended with `.shouldResume`: reactivate session, LiveKit resumes
- [ ] Handle `AVAudioSession.routeChangeNotification` for headphone/BT changes
- [ ] Create `Utilities/BackgroundKeepAlive.swift` — play inaudible tone every 90s if no audio activity (insurance against iOS killing background audio)
- [ ] App stays connected for 10+ minutes in background (manual test)

---

### Story 4: Reconnection State Machine
**As a** user  
**I want** the app to automatically reconnect when the connection drops  
**So that** I don't have to manually reopen the app after network issues

**Acceptance Criteria:**
- [ ] Create `ReconnectPolicy` struct: initial delay 0.5s, max delay 30s, backoff multiplier 1.5, max 20 attempts, ±30% jitter
- [ ] Create `Utilities/NetworkMonitor.swift` wrapping `NWPathMonitor`
- [ ] On disconnect: start reconnect loop with exponential backoff
- [ ] Each reconnect attempt: check `/health` first, only connect if server is up
- [ ] On network path change (WiFi↔cell): immediately attempt reconnect (reset backoff)
- [ ] Fetch fresh token on each reconnect attempt (token may have expired)
- [ ] Cancel reconnect loop on successful connect
- [ ] After max attempts exhausted: set state to `.failed`, fire local notification
- [ ] LiveKit SDK built-in reconnect handles ICE restarts; our loop is the fallback layer
- [ ] Server restart scenario: app reconnects within 30s of server coming back

---

### Story 5: Orb View + Main UI
**As a** user  
**I want** a minimal voice-first UI with an animated orb  
**So that** I can see connection state and speaking activity at a glance

**Acceptance Criteria:**
- [ ] Create `Views/VoiceView.swift` — main screen
- [ ] Create `Views/OrbView.swift` — animated orb using `TimelineView` + `Canvas`
- [ ] Orb states with colors: idle (slate gray, slow breathing), listening (slate gray, subtle pulse), thinking (soft blue, faster pulse), speaking (warm amber, audio-reactive radius), disconnected (red, static), muted (dimmed + slash overlay)
- [ ] Orb breathing animation: slow sine wave on radius (~1.5 Hz)
- [ ] Orb audio reactivity: map remote audio level to radius delta when speaking
- [ ] Detect Loom speaking via `RemoteTrackPublication.audioLevel > 0.01`
- [ ] "Loom" label below orb
- [ ] Tap orb → toggle mute (with haptic feedback)
- [ ] Bottom bar: mute button (🔇) + connection status dot (green/yellow/red)
- [ ] Status dot tappable → shows connection details (server, latency, room)
- [ ] Dark mode only
- [ ] SF Symbols for icons, system typography
- [ ] No onboarding flow — straight to voice screen

---

### Story 6: Settings Screen
**As a** user  
**I want** to configure server URL and see connection status  
**So that** I can manage the app's connection

**Acceptance Criteria:**
- [ ] Create `Views/SettingsView.swift` with NavigationStack
- [ ] CONNECTION section: server URL (editable text field), status indicator, room name, latency
- [ ] AUDIO section: push-to-talk toggle (stored in @AppStorage), mute state
- [ ] ABOUT section: app version, server version (from /health response)
- [ ] Settings gear icon on main VoiceView (top-right)
- [ ] Server URL changes trigger disconnect + reconnect
- [ ] Default server URL: `https://erics-macbook-pro.tail893ab1.ts.net:8089`
- [ ] Rewrite `SettingsManager.swift` to hold Loom Voice settings instead of VoxClaw's TTS settings

---

### Story 7: Mute + Push-to-Talk
**As a** user  
**I want** to mute my mic or use push-to-talk mode  
**So that** I have control over when Loom hears me

**Acceptance Criteria:**
- [ ] Mute toggle: `room.localParticipant.setMicrophone(enabled:)`
- [ ] Mute state reflected in orb (dimmed + slash)
- [ ] Haptic feedback on mute toggle (medium impact)
- [ ] Push-to-talk mode: long-press orb to unmute, release to mute
- [ ] Push-to-talk preference stored in `@AppStorage`
- [ ] When push-to-talk enabled, mic starts muted, orb shows "hold to talk" hint
- [ ] Works in background (push-to-talk only useful in foreground — that's fine)

---

### Story 8: Local Notifications
**As a** user  
**I want** to be notified when the connection drops  
**So that** I can reopen the app and reconnect

**Acceptance Criteria:**
- [ ] Create `Managers/NotificationManager.swift`
- [ ] Request notification permission on first launch
- [ ] Fire local notification when reconnect fails after 3 attempts
- [ ] Notification: title "Loom Voice", body "Connection lost. Tap to reconnect."
- [ ] Tapping notification opens app (default behavior)
- [ ] Don't spam — max 1 disconnect notification per 5 minutes
- [ ] Clear notification when app reconnects

---

### Story 9: Adapt Bonjour Discovery
**As a** user  
**I want** the app to auto-discover the Loom server on my LAN  
**So that** I don't need to manually enter the server URL

**Acceptance Criteria:**
- [ ] Adapt `PeerBrowser.swift` to browse for `_loom-voice._tcp` instead of `_voxclaw._tcp`
- [ ] Remove OpenClaw discovery (`_openclaw._tcp`)
- [ ] When a Loom server is discovered on LAN, offer it as connection option in Settings
- [ ] If no saved URL and Bonjour finds a server, auto-connect
- [ ] Bonjour is secondary to saved URL (don't override user's manual config)
- [ ] `Info.plist`: `NSLocalNetworkUsageDescription` and `NSBonjourServices` set
- [ ] Note: server-side Bonjour advertisement needs to be added separately (Python `zeroconf` package)

---

### Story 10: Rebrand + Polish
**As a** user  
**I want** the app to feel like Loom Voice, not VoxClaw  
**So that** it has its own identity

**Acceptance Criteria:**
- [ ] Bundle identifier: `com.ericday.loomvoice`
- [ ] App name: "Loom Voice"
- [ ] Replace VoxClaw app icon with Loom-themed icon (🪡 or custom — can use SF Symbols for dev)
- [ ] Update all references from "VoxClaw" to "Loom Voice" in code and UI
- [ ] Remove VoxClaw README, SKILL.md, docs/
- [ ] Update version to 1.0.0
- [ ] Remove VoxClaw GitHub references (About view links, etc.)
- [ ] Clean up any unused VoxClaw assets

---

## Story Order (Recommended)

1. **Story 1** — Strip to shell (foundation)
2. **Story 2** — LiveKit connection (core functionality)
3. **Story 3** — Background audio (the whole point)
4. **Story 4** — Reconnection (resilience)
5. **Story 5** — Orb UI (make it feel real)
6. **Story 6** — Settings (configuration)
7. **Story 7** — Mute + PTT (interaction)
8. **Story 8** — Notifications (polish)
9. **Story 10** — Rebrand (identity)
10. **Story 9** — Bonjour (nice-to-have, needs server change)

Stories 1-5 = MVP. Stories 6-10 = polish.

## Technical Notes

- **iOS 17+ target** (SwiftUI + Swift concurrency)
- **LiveKit SDK:** `https://github.com/livekit/client-sdk-swift` >= 2.12.1
- **No other dependencies** for MVP
- **Dark mode only**
- **Single user** — no auth beyond Tailscale network boundary
- **Estimated total:** ~1,500-2,000 lines of Swift
