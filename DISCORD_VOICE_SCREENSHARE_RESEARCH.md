# Discord Voice Channels & Screensharing Research
## Integration Analysis for Telegram Desktop Client

**Date:** 2025-11-06
**Purpose:** Research Discord's voice/video technology and analyze integration possibilities with Telegram Desktop (tdesktop)

---

## Table of Contents

1. [Discord Technology Stack](#1-discord-technology-stack)
2. [Telegram Current Implementation](#2-telegram-current-implementation)
3. [Comparative Analysis](#3-comparative-analysis)
4. [Integration Approaches](#4-integration-approaches)
5. [Technical Requirements](#5-technical-requirements)
6. [Recommendations](#6-recommendations)

---

## 1. Discord Technology Stack

### 1.1 Core Architecture

**Client-Server Model:**
- Discord uses a **client-server architecture** rather than peer-to-peer for voice/video
- Rationale: P2P becomes prohibitively expensive as participant count increases
- Supports up to **1,000 participants** in a single voice channel

**Technology Foundation:**
- Built on **WebRTC** (Web Real-Time Communication)
- Single **C++ media engine** shared across desktop, iOS, and Android
- Custom-built on top of WebRTC native library

### 1.2 Voice Server Components

Discord voice servers contain two main components:

#### A. Signaling Component
- Written in **Elixir**
- Fully controls the SFU (Selective Forwarding Unit)
- Generates stream identifiers and encryption keys
- Handles WebSocket connections for real-time events

#### B. Selective Forwarding Unit (SFU)
- Custom-built in **C++**
- Forwards audio/video traffic within channels
- Clients connect to media relay server (no direct P2P needed)
- Eliminates need for ICE (Interactive Connectivity Establishment)
- Keeps IP addresses secret from other participants

### 1.3 Backend Infrastructure

**Programming Languages:**
- **Elixir** - All signaling servers (allows code reuse)
- **Rust** - Performance-critical voice stack components
- **C++** - Media engine and SFU

**Connection Model:**
- Clients maintain **WebSocket connection** to Discord Gateway
- Receives events: group/channel updates, text messages, presence, voice state
- Gateway manages state synchronization

### 1.4 WebRTC Customizations

**Low-Level API Usage:**
- Uses `webrtc::Call` API (lower level than standard WebRTC)
- Creates separate send and receive streams
- Avoids SDP (Session Description Protocol) overhead
  - Standard SDP: ~10KB round-trip
  - Discord approach: ~1KB (voice server address, port, encryption keys, codec info, stream ID)

**ICE Elimination:**
- Since all clients connect to relay servers, ICE not needed
- More reliable connections behind NAT
- Enhanced privacy (IP addresses not exposed)

### 1.5 Codecs

**Audio:**
- **Opus codec** - Industry standard for VoIP
- Dynamic codec selection based on network conditions:
  - High quality: Opus at high bitrate
  - Medium quality: Opus at reduced bitrate
  - Poor connections: Legacy codecs or further reduced bitrate
- Real-time switching based on network telemetry

**Video:**
- **OpenH264** - Default codec (Cisco open-source)
- **AV1** - Next-generation codec (40% better encoding efficiency)
- Discord is first major platform to support AV1
- GPU-accelerated encoding/decoding (NVIDIA GeForce RTX support)

### 1.6 Performance Metrics

**Scale:**
- **2.6+ million** concurrent voice users
- **220+ Gbps** egress traffic
- **120+ million** packets per second
- **150+ million** monthly voice users

---

## 2. Telegram Current Implementation

### 2.1 Architecture Overview

**Dual Engine Support:**
- **WebRTC** - Modern implementation (primary)
- **TgVoip** - Telegram's proprietary legacy VoIP engine (fallback)
- **tgcalls** library - Main call implementation layer

**Call Types:**
1. **One-to-One Calls** - P2P with relay fallback
2. **Group Calls** - Server-mediated multiparty conferencing

### 2.2 Directory Structure

```
Telegram/SourceFiles/calls/
├── calls_call.h/cpp                    # One-to-one call logic
├── calls_controller.h/cpp              # Abstract controller interface
├── calls_controller_webrtc.h/cpp       # WebRTC implementation
├── calls_controller_tgvoip.h           # TgVoip legacy implementation
├── calls_instance.h/cpp                # Call instance manager
├── calls_panel.h/cpp                   # UI for one-to-one calls
├── calls_emoji_fingerprint.h/cpp       # Security verification
└── group/
    ├── calls_group_call.h/cpp          # Group call logic
    ├── calls_group_panel.h/cpp         # Group call UI
    ├── calls_group_viewport*.h         # Multi-participant rendering
    ├── calls_group_rtmp.h/cpp          # RTMP streaming
    ├── calls_group_message_encryption.h # E2E encryption
    └── ui/
        └── desktop_capture_choose_source.h # Screen share UI
```

### 2.3 Core Technologies

#### A. tgcalls Library
- **Location:** External dependency (not in tdesktop repo)
- **Repository:** https://github.com/MarshalX/tgcalls
- **Components:**
  - `tgcalls::GroupInstanceCustomImpl` - Group call engine
  - `tgcalls::VideoCaptureInterface` - Video capture abstraction
  - `tgcalls::VideoCodecName` - Codec enumeration
- **WebRTC Foundation:** Built on top of WebRTC native library
- **Language:** C++ with Python/JS bindings available

#### B. WebRTC Integration
- **Module:** `webrtc/webrtc_call_context.h`
- **Controller:** `WebrtcController` wraps call functionality
- **Device Management:** `Webrtc::DeviceResolver` for audio/video devices
- **Video Tracks:** `Webrtc::VideoTrack` manages video state

#### C. Network Infrastructure

**Connection Types:**
- UDP Relay endpoints (Telegram relay servers)
- STUN servers (NAT traversal)
- TURN servers (with authentication)
- P2P direct connections (when `enableP2P` flag is true)

**Server Configuration:**
- Endpoints provided via MTProto: `MTPPhoneConnection`, `MTPDphoneConnectionWebrtc`
- Each endpoint: IP (IPv4/IPv6), port, peer tag (16-byte ID)
- TURN auth: username="reflector", password=hex(peer_tag)

### 2.4 Encryption & Security

**End-to-End Encryption:**
- Diffie-Hellman key exchange before call establishment
- DH Config: 256-byte auth key, 32-byte SHA256
- Fingerprint: SHA1 hash of auth key
- Visual verification via emoji fingerprints (`calls_emoji_fingerprint.h`)

**Group Call Encryption:**
- Module: `calls_group_message_encryption.h`
- TdE2E integration for encrypted group calls
- Message serialization/deserialization with encryption

### 2.5 Signaling

**Protocol:** MTProto (Telegram's proprietary protocol)
- No SIP, no standard WebRTC signaling
- Signaling data: `MTPDupdatePhoneCallSignalingData`
- Callback: `sendSignalingData()` in controller interface
- Server-mediated even for P2P calls

### 2.6 Features

#### One-to-One Calls
- Voice and video support
- Camera sharing: `toggleCameraSharing()`
- Screen sharing: `toggleScreenSharing()`
- Audio device selection (microphone/speaker)
- Mute control, volume adjustment
- Echo cancellation strength adjustment
- Network type adaptation (WiFi, Cellular, Ethernet)

#### Group Calls
- Multi-user voice/video conferencing
- Participant tracking via SSRC (Synchronization Source)
- Video quality levels: Full, Medium, Low
- Adaptive quality based on bandwidth (max 16 medium quality streams)
- Viewport rendering with GPU acceleration (OpenGL) or CPU fallback (raster)
- Screen sharing with separate endpoint per participant
- RTMP streaming support for external broadcast

#### Screen Sharing
- **UI Component:** `desktop_capture_choose_source.h`
- Window/screen selection
- Optional audio capture with screen
- Source switching without call interruption
- Separate video endpoint (distinct from camera)

### 2.7 Audio Processing

**Features:**
- Microphone mute control
- Input/output volume control
- Echo cancellation
- Automatic Gain Control (AGC)
- Audio ducking (volume reduction during notifications)
- Device type resolution with fallback

**Statistics:**
- Traffic stats (data sent/received, packet loss)
- Signal quality (0-4 bars)
- VAD (Voice Activity Detection) levels
- Debug information for troubleshooting

---

## 3. Comparative Analysis

### 3.1 Similarities

| Feature | Discord | Telegram |
|---------|---------|----------|
| **WebRTC Foundation** | ✅ Yes | ✅ Yes |
| **Client-Server for Groups** | ✅ Yes | ✅ Yes |
| **Screen Sharing** | ✅ Yes | ✅ Yes |
| **Audio Codec (Opus)** | ✅ Yes | ✅ Likely (via WebRTC) |
| **E2E Encryption** | ✅ Optional | ✅ Yes (default) |
| **Adaptive Quality** | ✅ Yes | ✅ Yes |
| **GPU Acceleration** | ✅ Yes | ✅ Yes (OpenGL) |

### 3.2 Key Differences

| Aspect | Discord | Telegram |
|--------|---------|----------|
| **Architecture** | Always client-server (SFU) | P2P for 1-on-1, server for groups |
| **Max Participants** | 1,000 in voice channel | Varies by group type |
| **Signaling** | WebSocket + custom protocol | MTProto only |
| **ICE/NAT Traversal** | Not needed (relay-only) | STUN/TURN/ICE support |
| **IP Privacy** | Always hidden (via relay) | Optional (depends on P2P setting) |
| **Video Codecs** | OpenH264, AV1 | Via WebRTC (likely H264/VP8/VP9) |
| **SFU Implementation** | Custom C++ | Via tgcalls library |
| **Backend Language** | Elixir + Rust + C++ | Likely C++ (tgcalls) |
| **Legacy Fallback** | No | TgVoip proprietary engine |
| **RTMP Streaming** | Via bots/integrations | Native support |

### 3.3 Architectural Philosophy

**Discord:**
- **Simplicity:** All calls routed through SFU (no P2P complexity)
- **Scalability:** Optimized for large voice channels (1000+)
- **Privacy by Default:** IP addresses never exposed
- **Performance:** Custom low-overhead signaling (~1KB vs ~10KB SDP)

**Telegram:**
- **Flexibility:** P2P when possible, server when needed
- **Security:** E2E encryption by default with visual verification
- **Compatibility:** Dual engine (WebRTC + TgVoip) for broad support
- **Integration:** Deep MTProto integration with existing infrastructure

---

## 4. Integration Approaches

### 4.1 Approach A: Discord-Style SFU Layer

**Concept:** Add Discord-like SFU infrastructure to Telegram

**Implementation:**
1. **Create SFU Server Component**
   - Language: C++ (matches tgcalls) or Elixir (matches Discord)
   - Function: Forward audio/video streams between participants
   - Deploy: Telegram server infrastructure

2. **Modify tdesktop Client**
   - Add SFU connection mode in `calls_controller_webrtc.h`
   - Disable P2P/ICE for SFU mode
   - Update endpoint configuration to support SFU servers

3. **Extend MTProto**
   - New message types for SFU server addresses
   - Stream ID and encryption key exchange
   - Quality negotiation protocol

**Pros:**
- Better scalability for large groups
- Enhanced privacy (no IP exposure)
- Simpler NAT traversal

**Cons:**
- Requires server-side infrastructure changes
- Higher server bandwidth costs
- Complexity of deploying new server components
- Outside scope of tdesktop-only changes

**Feasibility:** ❌ **Not feasible** for tdesktop-only integration

---

### 4.2 Approach B: Enhanced Group Call Features

**Concept:** Improve existing Telegram group calls with Discord-like UX

**Implementation:**

#### Phase 1: UI/UX Enhancements
1. **Persistent Voice Channels**
   - Files: `calls_group_panel.h/cpp`
   - Feature: "Always-on" voice channels in groups
   - Allow joining/leaving without explicit calls

2. **Activity Status Indicators**
   - Module: `calls_group_members.h`
   - Show who's currently speaking (VAD already exists)
   - Visual indicators for mute/unmute, video on/off

3. **Quick Join/Leave**
   - Minimize call setup friction
   - One-click join from group chat UI

#### Phase 2: Performance Optimization
1. **Codec Optimization**
   - Implement dynamic Opus bitrate (if not already present)
   - Add AV1 codec support for video (via WebRTC update)

2. **Quality Adaptation**
   - Enhance existing quality switching in `calls_group_call.cpp`
   - More aggressive bandwidth adaptation

3. **Rendering Improvements**
   - Optimize `calls_group_viewport_opengl.h`
   - Better tile layout algorithms for many participants

#### Phase 3: Screen Share Enhancements
1. **Multi-Screen Share**
   - Already supports separate endpoints per user
   - Improve UI for selecting which screen to view

2. **Screen Share Quality Profiles**
   - Add presets: "Text/Code" (high res, low FPS) vs "Video" (lower res, high FPS)
   - Implement in `desktop_capture_choose_source.cpp`

**Pros:**
- Achievable within tdesktop codebase
- No server-side changes required
- Improves user experience immediately
- Builds on existing infrastructure

**Cons:**
- Doesn't fundamentally change architecture
- Limited by Telegram's server capabilities
- May not match Discord's scale

**Feasibility:** ✅ **Highly feasible** for tdesktop development

---

### 4.3 Approach C: Hybrid Bridge System

**Concept:** Create a bridge between Discord and Telegram voice systems

**Implementation:**

1. **Bridge Bot Architecture**
   - Separate process (not in tdesktop)
   - Connects to both Discord (via Discord.js or discord.py) and Telegram (via tgcalls)
   - Forwards audio/video between platforms

2. **tdesktop Integration Points**
   - Add "Join Discord Channel" feature in UI
   - Use existing group call infrastructure
   - Bridge appears as special participant

3. **Technical Components**
   - Audio mixing: Combine Discord streams for Telegram
   - Video transcoding: Handle codec differences
   - Signaling translation: Discord WebSocket ↔ Telegram MTProto

**Pros:**
- Actual Discord interoperability
- Could enable cross-platform communities

**Cons:**
- Very complex implementation
- Latency issues (double hop)
- Potential ToS violations (both platforms)
- Requires external service
- Quality degradation from transcoding

**Feasibility:** ⚠️ **Technically possible but not recommended**

---

### 4.4 Approach D: Inspired Feature Parity

**Concept:** Learn from Discord's design patterns, implement in Telegram's way

**Key Features to Adopt:**

#### 1. Voice Channel Paradigm
- **Discord Model:** Persistent channels users can join/leave freely
- **Telegram Implementation:**
  - Add "Voice Channel" topic type in Forums/Groups
  - Modify `calls_group_call.h` to support "persistent" mode
  - No explicit "start call" - channel always ready
  - File: `Telegram/SourceFiles/data/data_group_call.h`

#### 2. Improved Audio Quality
- **Discord Model:** Dynamic Opus bitrate, excellent quality
- **Telegram Implementation:**
  - Audit current Opus configuration in tgcalls usage
  - Expose bitrate controls in `settings/settings_calls.cpp`
  - Add quality presets: "Auto", "High Quality", "Data Saver"

#### 3. Screen Share Optimization
- **Discord Model:** Smooth screen sharing with quality presets
- **Telegram Implementation:**
  - Add quality profiles in `desktop_capture_choose_source.cpp`
  - Presets:
    - "Presentation" - 1080p @ 5fps
    - "Video Playback" - 720p @ 30fps
    - "Gaming" - 1080p @ 60fps (if supported)
  - Implement adaptive FPS based on content change detection

#### 4. Noise Suppression
- **Discord Model:** Krisp AI noise suppression
- **Telegram Implementation:**
  - WebRTC has built-in noise suppression
  - Expose controls in `settings_calls.cpp`
  - Add "Noise Suppression Level": Off, Low, Medium, High
  - Already have echo cancellation strength setting

#### 5. Visual Indicators
- **Discord Model:** Clear speaking indicators, connection quality
- **Telegram Implementation:**
  - Enhance `calls_group_members.h` with better indicators
  - Speaking: Animated ring around avatar (VAD already exists)
  - Quality: Color-coded (green/yellow/red) based on signal bars
  - Bandwidth: Show current bitrate for debug

**Pros:**
- Best of both worlds approach
- Maintains Telegram's security/privacy model
- Fully implementable in tdesktop
- No server-side dependencies

**Cons:**
- Doesn't replicate Discord exactly
- Some features may feel "bolted on"

**Feasibility:** ✅ **Most practical approach**

---

## 5. Technical Requirements

### 5.1 For Enhanced Group Calls (Approach B/D)

#### Code Modules to Modify

1. **Group Call Core** (`calls/group/calls_group_call.cpp`)
   - Add persistent channel mode
   - Implement auto-reconnect logic
   - Enhanced quality adaptation

2. **Group Call UI** (`calls/group/calls_group_panel.cpp`)
   - Redesign UI for "always-on" paradigm
   - Quick join/leave controls
   - Better participant list with indicators

3. **Settings** (`settings/settings_calls.cpp`)
   - Add quality presets
   - Noise suppression levels
   - Screen share quality profiles

4. **Screen Capture** (`calls/group/ui/desktop_capture_choose_source.cpp`)
   - Quality profile selection UI
   - FPS/resolution presets
   - Content type detection (future)

5. **Member Management** (`calls/group/calls_group_members.cpp`)
   - Enhanced visual indicators
   - Speaking animations
   - Connection quality display

#### External Dependencies

1. **tgcalls Library**
   - May need updates for advanced features
   - Check version compatibility
   - Potential custom fork for Telegram-specific features

2. **WebRTC**
   - Ensure recent version for AV1 support
   - Noise suppression module enabled
   - Adaptive bitrate capabilities

3. **Qt Framework**
   - OpenGL for GPU-accelerated rendering
   - QML for dynamic UI elements
   - Multimedia for device enumeration

### 5.2 Platform Considerations

**Linux:**
- PipeWire/PulseAudio for audio
- X11/Wayland screen capture APIs
- Current implementation: Likely already supported

**Windows:**
- WASAPI for audio
- DXGI Desktop Duplication for screen capture
- Current implementation: Likely already supported

**macOS:**
- Core Audio for audio
- CGDisplayStream for screen capture
- Current implementation: Likely already supported

### 5.3 Testing Requirements

1. **Scale Testing**
   - Group calls with 10, 50, 100+ participants
   - Monitor CPU/memory usage
   - Bandwidth consumption

2. **Quality Testing**
   - Audio quality at various bitrates
   - Video quality with different codecs
   - Screen share smoothness

3. **Network Resilience**
   - Packet loss scenarios (1%, 5%, 10%)
   - High latency conditions (100ms, 500ms)
   - Bandwidth constraints

4. **Cross-Platform**
   - Test on all supported OSes
   - Verify feature parity
   - UI consistency

---

## 6. Recommendations

### 6.1 Recommended Approach

**Primary: Approach D - Inspired Feature Parity**

**Rationale:**
1. **Feasibility:** Fully achievable within tdesktop codebase
2. **No Server Changes:** Works with existing Telegram infrastructure
3. **User Value:** Delivers Discord-like UX improvements
4. **Maintainability:** Follows existing code patterns
5. **Security:** Preserves Telegram's E2E encryption

### 6.2 Implementation Phases

#### Phase 1: Foundation (1-2 months)
**Goal:** Improve core group call experience

1. **Audio Quality Enhancements**
   - Audit and optimize Opus configuration
   - Add quality presets in settings
   - Implement adaptive bitrate (if not present)
   - **Files:** `settings/settings_calls.cpp`, controller implementations

2. **Visual Indicators**
   - Speaking animations (use existing VAD)
   - Connection quality colors
   - Bandwidth display (debug mode)
   - **Files:** `calls/group/calls_group_members.cpp`, `calls_group_panel.cpp`

3. **Noise Suppression Controls**
   - Expose WebRTC noise suppression
   - Add UI controls for levels
   - **Files:** `settings/settings_calls.cpp`, controller interface

#### Phase 2: Screen Share Improvements (1 month)
**Goal:** Discord-level screen sharing

1. **Quality Profiles**
   - Implement presets (Presentation, Video, Gaming)
   - FPS/resolution control
   - **Files:** `desktop_capture_choose_source.cpp`

2. **UI Enhancements**
   - Better source selection
   - Preview window
   - One-click share last used source

#### Phase 3: Voice Channel UX (2-3 months)
**Goal:** Persistent voice channel experience

1. **Persistent Channel Mode**
   - "Always ready" voice channels
   - Quick join/leave (one click)
   - Auto-reconnect on disconnect
   - **Files:** `calls/group/calls_group_call.cpp`, `data/data_group_call.h`

2. **UI Redesign**
   - Channel list in group sidebar
   - Live participant count
   - Join state persistence
   - **Files:** `calls/group/calls_group_panel.cpp`, main window integration

3. **Performance Optimization**
   - Lazy loading for large participant lists
   - Optimized rendering for many tiles
   - **Files:** `calls/group/calls_group_viewport*.cpp`

#### Phase 4: Advanced Features (2-3 months)
**Goal:** Differentiation and polish

1. **Multi-Camera Support**
   - Switch between cameras without reconnect
   - Picture-in-picture for dual camera

2. **Advanced Codec Support**
   - AV1 for video (update WebRTC)
   - Codec negotiation improvements

3. **Streaming Enhancements**
   - Better RTMP integration
   - Multi-platform streaming

### 6.3 Risks & Mitigation

| Risk | Impact | Mitigation |
|------|--------|------------|
| **tgcalls Limitations** | High | Fork and modify tgcalls if needed |
| **Server-Side Constraints** | Medium | Design features to work within limits |
| **Cross-Platform Bugs** | Medium | Extensive testing on all platforms |
| **Performance Regression** | High | Benchmarking, profiling, optimization |
| **UI/UX Complexity** | Low | User testing, iterative design |

### 6.4 Success Metrics

**Quantitative:**
- Group call participation increase by 30%
- Average call duration increase by 20%
- Screen share usage increase by 50%
- CPU usage decrease or stable
- Fewer connection failures (<1% drop rate)

**Qualitative:**
- User feedback: "Easier to use"
- Comparison: "As good as Discord"
- Feature requests: Addressed backlog items

### 6.5 Non-Goals

**What NOT to do:**
1. ❌ Change Telegram's server architecture (outside tdesktop scope)
2. ❌ Remove P2P for one-to-one calls (security feature)
3. ❌ Replace MTProto signaling (core protocol)
4. ❌ Build Discord bridge (complexity, ToS issues)
5. ❌ Compromise E2E encryption (security principle)

---

## 7. Scalability Assessment for Large Teams (100+ Users)

### 7.1 Current Telegram Group Call Limits

**Server-Side Constraints:**

| Metric | Limit | Code Reference |
|--------|-------|----------------|
| **Total Participants** | 1,000 viewers/listeners | Server-enforced |
| **Active Broadcasters** | 30 simultaneous video/audio | `data_group_call.cpp:537` (`unmutedVideoLimit`) |
| **Client Video Rendering** | 16 medium quality streams | `calls_group_call.cpp:61` (`kMaxMediumQualities`) |
| | 4 full quality streams | Equivalent to 16 medium |

**Key Finding:** The 30 active broadcaster limit is **server-side** and controlled by `unmutedVideoLimit` field from Telegram's backend. This cannot be changed in tdesktop client code.

### 7.2 100-Person Team Requirements Analysis

#### Scenario 1: All-Hands Meetings (Few Speakers)
**Use Case:** Company-wide announcements, presentations

- **Speakers:** 5-10 people (presenters, leadership)
- **Listeners:** 90-95 people
- **Verdict:** ✅ **WORKS WELL**
  - Well within 30 active broadcaster limit
  - All 100 within 1,000 total participant limit
  - Recommended: Designate speakers, mute others

#### Scenario 2: Team Standup (Moderate Interaction)
**Use Case:** Daily standups, sprint planning

- **Active Speakers:** 15-25 people (rotating)
- **Listeners:** 75-85 people
- **Verdict:** ✅ **WORKS** (with management)
  - Within 30 broadcaster limit if managed
  - Use push-to-talk or hand-raise features
  - Moderator unmutes speakers in sequence

#### Scenario 3: Open Discussion (High Interaction)
**Use Case:** Brainstorming sessions, open debates

- **Potential Speakers:** 50-100 people (anyone can speak)
- **Simultaneous Speakers:** 30+ at peak
- **Verdict:** ⚠️ **BOTTLENECK**
  - **Problem:** 30 active broadcaster hard limit
  - Only 30 can broadcast simultaneously
  - Others blocked from unmuting/video when limit reached
  - Results in "take turns" situation

#### Scenario 4: Persistent Voice Channels (Discord-Style)
**Use Case:** "Always-on" channels where team members drop in/out

- **Typical Occupancy:** 20-40 people in channel
- **Active Speakers:** Variable (5-15 typically)
- **Verdict:** ✅ **WORKS** (most of the time)
  - Usually under 30 active limit
  - May hit limit during peak times
  - Need intelligent audio routing (PTT recommended)

### 7.3 Comparison: Discord vs Telegram at 100-Person Scale

| Capability | Discord | Telegram (Current) |
|------------|---------|-------------------|
| **Total Capacity** | 1,000 participants | 1,000 participants |
| **Simultaneous Audio Broadcasters** | 1,000 (all can speak) | 30 (server limit) |
| **Simultaneous Video Broadcasters** | 1,000 | 30 (server limit) |
| **Client Rendering Limit** | ~25-50 video tiles | 16 medium quality streams |
| **100-Person Open Discussion** | ✅ Supported | ❌ Bottleneck (30 limit) |
| **100-Person Presentation** | ✅ Supported | ✅ Supported |

### 7.4 Infrastructure Sufficiency Assessment

**For a 100-person growing team:**

#### ❌ Current Infrastructure (Approach D) - INSUFFICIENT for Full Interactivity

**Limitations:**
1. **30 active broadcaster ceiling** - Cannot be overcome without server changes
2. **Blocks "everyone can speak" scenarios** - Critical for collaborative teams
3. **Not scalable beyond 30 simultaneous speakers** - Growing team will hit this wall
4. **Code Evidence:**
   ```cpp
   // calls/group/calls_group_call.cpp:2691
   if (real && activeVideoSendersCount() >= real->unmutedVideoLimit()) {
       // User blocked from enabling video
   }
   ```

#### ⚠️ Workarounds (Tactical Solutions)

1. **Push-to-Talk (PTT) Enforcement**
   - Prevents accidental broadcaster limit hits
   - Requires discipline
   - Code location: Can enhance in `calls_controller.h`

2. **Moderator-Controlled Unmuting**
   - Manual management of who can speak
   - Scalability issue: requires active moderation
   - Already supported via group call permissions

3. **Split Into Multiple Smaller Calls**
   - Divide 100-person team into 4x 25-person groups
   - Defeats purpose of unified team communication
   - Organizational overhead

4. **Hybrid Model: Voice-Only for Most**
   - 30 video slots for presenters
   - Remaining 70 use audio-only (still counts toward 30 if broadcasting)
   - Still hits 30 active broadcaster limit

**Conclusion:** Workarounds are **tactical patches**, not strategic solutions.

#### ✅ Required Solution: Approach A (Discord-Style SFU) - NECESSARY

**Why Approach A is Required:**

To truly support 100+ simultaneous speakers, you need:

1. **Server-Side SFU Implementation**
   - Remove 30 broadcaster limit
   - Scale to 100+ active audio/video streams
   - **Cannot be done in tdesktop alone**

2. **Backend Infrastructure Changes**
   - Deploy SFU servers (Elixir/Rust/C++ as Discord does)
   - Modify Telegram server to increase `unmutedVideoLimit`
   - Update MTProto to support higher limits

3. **Combined Approach**
   - Implement Approach A (server-side SFU) **AND**
   - Implement Approach D (client-side UX improvements)
   - Timeline: 6-12 months for full implementation

### 7.5 Recommendation for 100-Person Team

#### Short-Term (0-3 months) - Use Current Infrastructure with Constraints

1. **Accept 30 broadcaster limit**
2. **Implement Approach D UX improvements** (from research)
   - Better quality controls
   - Speaking indicators
   - Screen share optimization
3. **Establish team protocols**
   - Push-to-talk for large meetings
   - Moderator-managed speaker queue
   - Split into smaller sub-teams when needed

**Viability:** ⚠️ **Partial** - Works for presentations, limited interaction

---

#### Long-Term (3-12 months) - Server Infrastructure Upgrade

1. **Implement Approach A: Discord-Style SFU**
   - Work with Telegram server team (or fork server code)
   - Deploy custom SFU infrastructure
   - Increase `unmutedVideoLimit` to 100+

2. **Combine with Approach D Client Improvements**
   - Full Discord-like experience
   - 100+ simultaneous speakers supported
   - Client handles rendering 16-30 videos, but all can speak

**Viability:** ✅ **Full Solution** - Requires server-side development

---

#### Alternative: Use Discord for Large Interactive Meetings

**Pragmatic Assessment:**

If the requirement is **100 people in open discussion RIGHT NOW**:
- Telegram's current infrastructure cannot support this
- Discord can support this today
- Consider using Discord for specific large interactive sessions
- Use Telegram for everything else (messaging, smaller calls, security)

**Hybrid Approach:**
- Telegram: Daily communication, small team calls (<30 active)
- Discord: Large all-hands, open discussions (100+ active)
- Timeline: Immediate (no development needed)

### 7.6 Updated Feasibility Matrix

| Approach | 100-Person Support | Timeline | Feasibility |
|----------|-------------------|----------|-------------|
| **A: Discord-Style SFU** | ✅ Full (100+ speakers) | 6-12 months | ⚠️ Requires server dev |
| **B: Enhanced Group Calls** | ⚠️ Partial (30 speakers) | 2-4 months | ✅ tdesktop-only |
| **C: Hybrid Bridge** | ❌ Complex, ToS issues | 4-6 months | ❌ Not recommended |
| **D: Inspired Feature Parity** | ⚠️ Partial (30 speakers) | 3-6 months | ✅ tdesktop-only |
| **E: Use Discord Directly** | ✅ Full (1000+ speakers) | Immediate | ✅ No development |

---

## 8. Conclusion

Discord's voice and screensharing success comes from:
1. **Simplified architecture** (SFU eliminates P2P complexity)
2. **Optimized signaling** (custom protocol, 1KB vs 10KB)
3. **Excellent codecs** (Opus, AV1)
4. **User-first UX** (persistent channels, quick join/leave)
5. **Unlimited active speakers** (all 1,000 participants can broadcast)

Telegram can adopt Discord's best UX patterns while maintaining its core principles:
- **Security:** E2E encryption
- **Privacy:** Optional P2P, IP protection
- **Flexibility:** Dual engine support

### Final Recommendations Based on Team Size

**For teams <30 active participants:**
- **Recommended path:** Implement Approach D (Inspired Feature Parity)
- Focus on UX improvements within tdesktop
- No server changes required
- Timeline: 3-6 months

**For teams with 100+ active participants:**
- **Current infrastructure:** ❌ INSUFFICIENT (30 broadcaster limit)
- **Required solution:** Approach A (Discord-Style SFU) + server upgrades
- **Alternative:** Use Discord for large interactive meetings, Telegram for everything else
- **Timeline for full solution:** 6-12 months (requires server-side development)

### Critical Constraint Discovered

The research initially focused on client-side improvements, but **scalability analysis reveals a fundamental server-side bottleneck**: Telegram's 30 active broadcaster limit (vs Discord's 1,000). This limit is hardcoded server-side (`unmutedVideoLimit`) and cannot be changed in tdesktop alone.

**Bottom line:** Client-side improvements (Approach D) enhance UX but don't solve the 100-person concurrent speaker requirement. That requires server infrastructure changes (Approach A).

---

## 9. References

### Documentation
- Discord Engineering Blog: WebRTC at Scale
- Telegram API Documentation: Calls
- WebRTC Official Documentation
- tgcalls GitHub Repository

### Code Locations
- **tdesktop:** `/home/user/tdesktop/Telegram/SourceFiles/calls/`
- **tgcalls:** https://github.com/MarshalX/tgcalls
- **WebRTC:** Third-party dependency

### Key Files for Modification
1. `calls/group/calls_group_call.cpp` - Core group call logic
2. `calls/group/calls_group_panel.cpp` - Group call UI
3. `calls/group/ui/desktop_capture_choose_source.cpp` - Screen share
4. `calls/group/calls_group_members.cpp` - Participant management
5. `settings/settings_calls.cpp` - Call settings UI
6. `calls/calls_controller.h` - Controller interface (add features)
7. `webrtc/webrtc_call_context.h` - WebRTC integration

---

**End of Research Document**
