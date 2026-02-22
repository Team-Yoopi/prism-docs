---
title: Module Strategy
---

# Prism Video Player - Module Strategy

> Last updated: 2026-02-08
>
> This document covers the business model, pricing strategy, and technical
> implementation of Prism's modular asset store packages.

## Design Principles

1. **Core must be genuinely useful standalone.** Not a crippled demo. It replaces
   Unity's VideoPlayer with something better. Customers who only need MP4/HLS
   get full value without buying add-ons.

2. **Modules map to distinct customer segments.** Each module addresses a different
   use case and audience. Nobody should need to buy two modules for one workflow.

3. **No nickel-and-diming.** Fewer chunky modules > many thin slices. Bundle
   discount available for customers who want everything.

4. **Technical modularity = business modularity.** Each module is a separate
   Asset Store package with its own native DLL. Missing DLL = feature unavailable,
   not compile error. Zero coupling between modules.

5. **No LGPL.** This is the key competitive advantage over AVPro Video and similar
   products that depend on FFmpeg. Every native component uses OS-provided APIs
   (Media Foundation, Winsock, SChannel) — all built-in and royalty-free.

---

## Module Overview

### Prism Player (Core) — $15-20

**Customer:** Any Unity developer who needs video playback.

**Value prop:** "Unity's built-in VideoPlayer is unreliable and limited. Prism Player
uses hardware-accelerated OS media frameworks for fast, correct playback with proper
HLS support. No FFmpeg, no LGPL headaches."

**Includes:**
- MP4, MOV, WMV playback via Windows Media Foundation
- HLS streaming via Windows Media Engine (full adaptive bitrate)
- FMP4 (fragmented MP4) live stream playback
- Hardware-accelerated H.264/HEVC decode
- Texture2D + RenderTexture output
- Stereo audio via AudioSource
- Play / pause / seek / loop
- `IPrismPlayer` C# interface
- `PrismVideoPlayer` MonoBehaviour (high-level, handles backend selection)
- `NativeFramePump` shared utility (texture upload, audio routing)
- Unity VideoPlayer fallback for unsupported platforms
- Works in Editor + Standalone builds

**Native DLL:** `prism_player.dll` (Windows), future: `.dylib` (macOS), `.so` (Android)

**Why this is the core:** Every other module needs video output. This provides
the shared player infrastructure that modules plug into.

---

### Platform Resolver — $20-30

**Customer:** Developers building "watch together" features, media galleries,
or any app where users paste URLs from video platforms.

**Value prop:** "Paste a YouTube or Twitch URL and it just plays. No server,
no API keys needed for basic playback. Quality selection from 360p to 4K."

**Includes:**
- YouTube URL resolution (via bundled yt-dlp)
- Twitch live stream resolution
- Vimeo, Dailymotion, Twitter/X, TikTok, Facebook
- 1800+ supported sites (everything yt-dlp supports)
- Quality selection (360p / 480p / 720p / 1080p / 1440p / 4K)
- Live stream detection
- Metadata extraction (title, duration, thumbnail URL)
- Async resolution (non-blocking)
- Bundled `yt-dlp.exe` (no user setup required)

**Native DLL:** `prism_resolver.dll`

**Requires:** Prism Player (Core)

**Notes:**
- yt-dlp is Unlicense (public domain equivalent) — no licensing concerns
- Resolution happens locally, no proxy server needed
- Some platforms (YouTube) may require cookies for age-restricted content
- Future: OAuth integration for Twitch subscriber-only streams

---

### Live Stream Pro — $45-60

**Customer:** VR social app developers, live event platforms, custom streaming
applications. Anyone who needs to receive live video from RTMP/RTSP/WebSocket
sources without running a gateway server.

**Value prop:** "Native low-latency live streaming for Unity. Supports RTMP,
RTSP, WebSocket-FLV, MPEG-TS, and WebRTC — all decoded in hardware with no
FFmpeg and no gateway server. VRCDN-ready out of the box."

**Includes:**
- RTMP subscriber (connect to any RTMP server)
- RTSP/RTSPT client (TCP interleaved, low latency)
- WebSocket FLV receiver (wss:// + ws://)
- MPEG-TS over HTTP receiver
- WebRTC/WHEP ultra-low latency receiver (100-500ms)
- VRCDN native support (all 5 format endpoints)
- H.264 + AAC decode via Media Foundation (hardware accelerated)
- H.264 + Opus decode (WebRTC path)
- Auto-reconnection on disconnect
- No gateway server required
- No FFmpeg dependency, no LGPL

**Native DLL:** `prism_live.dll`

**Requires:** Prism Player (Core)

**Why WebRTC is bundled here (not separate):**
1. Customers who need RTMP almost always also need RTSP and WebRTC
2. All protocols share 80% of the decode/output pipeline internally
3. Splitting further creates a "nickel and dime" perception
4. One module = one purchase decision for "I need live streaming"

**VRCDN Endpoint Coverage:**

| VRCDN Format | URL | Handled By |
|---|---|---|
| FMP4 | `https://stream.vrcdn.live/live/{user}.live.mp4` | Core (Media Engine) |
| FLV/WebSocket | `wss://stream.vrcdn.live/live/{user}.flv` | Live Stream Pro |
| RTSPT | `rtspt://stream.vrcdn.live/live/{user}` | Live Stream Pro |
| RTMP | `rtmp://stream.vrcdn.live/live/{user}` | Live Stream Pro |
| MPEG-TS | `https://stream.vrcdn.live/live/{user}.live.ts` | Live Stream Pro |

---

### Spatial Audio — $15-20

**Customer:** VR developers who need immersive audio from video sources.

**Value prop:** "5.1/7.1 surround sound from any Prism video source. Virtual
speaker placement for VR, per-channel access for custom audio routing."

**Includes:**
- 5.1 and 7.1 surround sound decode
- Virtual speaker placement (SurroundSpeakerSetup component)
- Per-channel sample access API (`IPrismAudioAccess`)
- Stereo downmix (ITU-R BS.775 standard)
- Raw audio sample callback for custom processing
- Works with any Prism player backend

**Native DLL:** None (pure C# implementation)

**Requires:** Prism Player (Core)

---

### Prism Complete Bundle — $65-85

All four modules at 20-25% discount.

**Customer:** Anyone who wants everything. VR social platforms, media-heavy
applications, live event tools.

---

## Pricing Rationale

| Module | Price | Justification |
|---|---|---|
| Core | $15-20 | Low barrier to entry. Competes with free alternatives on quality. Gets users into the ecosystem. |
| Resolver | $20-30 | Clear value: "paste URL, it plays." Worth it for any social/media app. |
| Live Stream | $45-60 | High-value niche. No competition at this price without FFmpeg/LGPL. AVPro charges $300+ and requires FFmpeg. |
| Spatial Audio | $15-20 | Small add-on. Pure C#, low maintenance cost. VR devs will buy it alongside Live Stream. |
| Bundle | $65-85 | ~25% savings vs individual. Anchors perceived value of individual modules. |

**Competitive positioning:**
- Unity VideoPlayer: Free but broken (no HLS on many platforms, poor codec support)
- AVPro Video: $300+ and requires FFmpeg (LGPL, large binary, complex licensing)
- Prism: Affordable modules, no FFmpeg, no LGPL, hardware-only decode

---

## Technical Implementation

### Unity Package Structure

Each module is a separate UPM package:

```
# Core (always installed)
com.teamyoopi.prism

# Optional modules (separate Asset Store packages)
com.teamyoopi.prism.resolver
com.teamyoopi.prism.livestream
com.teamyoopi.prism.spatialaudio
```

### File Layout

```
prism-unity-plugin/                          # Core package
├── package.json                             # com.teamyoopi.prism
├── Runtime/
│   ├── Core/
│   │   ├── IPrismPlayer.cs                 # Universal interface
│   │   ├── PrismPlayerFactory.cs           # Module auto-detection
│   │   ├── PrismVideoPlayer.cs             # High-level component
│   │   └── NativeFramePump.cs              # Shared frame/audio utility
│   ├── Native/Windows/
│   │   ├── WindowsMediaEngineBridge.cs     # P/Invoke for prism_player.dll
│   │   └── WindowsMediaEnginePlayer.cs     # IPrismPlayer impl
│   ├── Audio/
│   │   └── PrismAudioMode.cs
│   └── UI/
│       └── VideoPlayerUI.cs
└── Plugins/Windows/x86_64/
    └── prism_player.dll

prism-resolver-module/                       # Platform Resolver package
├── package.json                             # com.teamyoopi.prism.resolver
├── Runtime/
│   ├── ResolverBridge.cs                   # P/Invoke for prism_resolver.dll
│   └── PlatformResolver.cs                 # High-level resolution API
└── Plugins/Windows/x86_64/
    ├── prism_resolver.dll
    └── yt-dlp.exe

prism-live-module/                           # Live Stream Pro package
├── package.json                             # com.teamyoopi.prism.livestream
├── Runtime/
│   ├── LiveStreamBridge.cs                 # P/Invoke for prism_live.dll
│   └── WindowsLivePlayer.cs               # IPrismPlayer impl (thin, uses NativeFramePump)
└── Plugins/Windows/x86_64/
    └── prism_live.dll

prism-spatial-audio-module/                  # Spatial Audio package
├── package.json                             # com.teamyoopi.prism.spatialaudio
├── Runtime/
│   ├── SurroundSpeakerSetup.cs
│   └── SpatialAudioRouter.cs
└── (no native DLLs)
```

### Module Detection Pattern

Core's `PrismPlayerFactory` probes for available modules without compile-time
references. This is the same pattern already used for WebRTC detection.

```csharp
public static class PrismPlayerFactory
{
    private static bool _nativeAvailable;
    private static bool _liveAvailable;
    private static bool _resolverAvailable;

    private static void CheckAvailability()
    {
        // Each check wraps DllImport in try/catch for DllNotFoundException.
        // Missing DLL = module not installed = false.
        // No compile errors, no assembly references needed.

        _nativeAvailable = TryNativeCheck();    // prism_player.dll
        _liveAvailable = TryLiveCheck();        // prism_live.dll
        _resolverAvailable = TryResolverCheck(); // prism_resolver.dll
    }
}
```

### URL Routing After Modularization

The `NormalizeVrcdnUrl()` method is **deleted entirely**. All URLs pass through
as-is. Backend selection is based on URL scheme + module availability:

```csharp
private PrismPlayerBackend SelectBackend(string url)
{
    // Live protocols → Live module
    if (IsLiveProtocol(url))  // rtmp://, rtspt://, rtsp://, wss://, whep://
    {
        if (PrismPlayerFactory.IsLiveAvailable)
            return PrismPlayerBackend.Live;
        else
            LogError("Live Stream Pro module required for this URL");
    }

    // MPEG-TS over HTTP → Live module
    if (url.EndsWith(".ts") || url.EndsWith(".live.ts"))
    {
        if (PrismPlayerFactory.IsLiveAvailable)
            return PrismPlayerBackend.Live;
    }

    // Native formats → Core player
    if (IsNativeFormat(url))  // .mp4, .mov, .m3u8, .live.mp4, etc.
        return PrismPlayerBackend.PlatformNative;

    // Platform URL → Resolve first, then re-route
    if (PrismPlayerFactory.IsResolverAvailable && NeedsResolution(url))
        return PrismPlayerBackend.PlatformNative; // Will resolve first

    return PrismPlayerFactory.GetBestAvailableBackend();
}
```

### Cross-Module Communication

Modules never reference each other directly. Communication happens through:

1. **Core interfaces** (`IPrismPlayer`) that all modules implement
2. **PrismPlayerFactory** availability checks (try/catch DllImport)
3. **PrismVideoPlayer** as the orchestrator that selects backends

If a customer installs Core + Live Stream (no Resolver), YouTube URLs show an
error. If they install Core + Resolver (no Live Stream), RTMP URLs show an error.
Each module fails gracefully with a clear message about what's needed.

---

## Future Modules (Potential)

| Module | Description | Priority |
|---|---|---|
| **Recording / Capture** | Record video from any Prism source to MP4 | Medium |
| **Multi-View** | Display multiple streams simultaneously | Medium |
| **DRM / Encryption** | Widevine / FairPlay / custom DRM support | Low |
| **Transcoding** | Real-time format conversion (e.g., RTMP → HLS) | Low |
| **Analytics** | Playback metrics, buffering stats, QoE scoring | Low |

New modules follow the same pattern: separate package, own DLL (if native code
needed), detected at runtime, no compile-time coupling.

---

## Quest / Android Strategy

MPEG-TS is specifically noted as the Quest format for VRCDN. Android native
decode uses MediaCodec instead of Media Foundation. The plan:

1. Core C# layer is already platform-agnostic
2. Each module ships platform-specific native binaries:
   ```
   Plugins/
   ├── Windows/x86_64/prism_live.dll
   ├── Android/arm64-v8a/libprism_live.so    # Future
   └── macOS/prism_live.bundle               # Future
   ```
3. `NativeFramePump` works the same regardless of platform
4. Android live plugin would use:
   - Plain sockets (POSIX) for RTMP/RTSP/WS
   - OpenSSL or BoringSSL for TLS (Android doesn't have SChannel)
   - MediaCodec for H.264/AAC decode
   - Same demux code (FLV, TS, RTP parsers are pure C, platform-agnostic)

---

## Revenue Projections & Priorities

**Phase 1 (ship first):** Core + Live Stream Pro
- These solve the immediate need (VRCDN support without gateway)
- Live Stream Pro is the highest-value module

**Phase 2:** Platform Resolver
- Split from prism_native_win.dll into separate package
- Broad market appeal

**Phase 3:** Spatial Audio
- Extract from current WebRTC player into standalone module
- VR-focused upsell

**Phase 4:** Bundle + marketing push
- All modules available, bundle pricing active
- Marketing: "Like AVPro but affordable, no FFmpeg, no LGPL"
