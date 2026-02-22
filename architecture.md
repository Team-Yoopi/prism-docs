---
title: Architecture
---

# Prism Video Player - System Architecture

> Last updated: 2026-02-08

Prism Video Player is a modular, plugin-based Unity video player. It uses native OS media frameworks (no FFmpeg) and sells as a Unity Asset Store package with optional add-on modules.

## Repository Map

```
cheeks/yoopi/
├── prism-unity-plugin/         Unity C# package (core + module C# scripts)
├── prism-video-player/         Native Media Foundation player (prism_player.dll)
├── prism-common/               Shared static library (MF decoders, ringbuffers)
├── prism-live-plugin/          Unified live stream plugin (prism_live.dll)
├── prism-ytdlp-plugin/         URL resolver (prism_resolver.dll)
├── prism-docs/                 This documentation
├── prism-gateway/              MediaMTX gateway config (deprecated, removing)
└── prism-ffmpeg-plugin/        FFmpeg decoder (deprecated, replaced by native)
```

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Unity Application                            │
├─────────────────────────────────────────────────────────────────────┤
│                      prism-unity-plugin                             │
│                                                                     │
│   PrismVideoPlayer           High-level component (URL routing,     │
│   (MonoBehaviour)            backend selection, lifecycle)          │
│         │                                                           │
│         ├─── PrismPlayerFactory      Auto-detect available modules  │
│         │                                                           │
│         ├─── NativeFramePump         Shared texture upload, audio   │
│         │    (utility class)         routing, flip buffer, downmix  │
│         │                            Written ONCE, used by all      │
│         │                                                           │
│   ┌─────┴──────────────────────────────────────────────────────┐    │
│   │                    IPrismPlayer backends                    │    │
│   │                                                             │    │
│   │  WindowsMediaEnginePlayer    MP4/HLS/FMP4 via Media Engine  │    │
│   │  (Core)                      Uses NativeFramePump           │    │
│   │                                                             │    │
│   │  WindowsLivePlayer           All live protocols             │    │
│   │  (Live Stream Pro module)    Uses NativeFramePump           │    │
│   │                                                             │    │
│   │  PrismPlayer                 Unity VideoPlayer wrapper      │    │
│   │  (Core fallback)             For platforms without native   │    │
│   └─────────────────────────────────────────────────────────────┘    │
├─────────────────────────────────────────────────────────────────────┤
│                         Native DLLs                                 │
│                                                                     │
│   prism_player.dll         Media Foundation / Media Engine          │
│   (Core)                   MP4, MOV, WMV, HLS, FMP4                │
│                                                                     │
│   prism_live.dll           All live streaming protocols             │
│   (Live Stream Pro)        WebRTC, RTMP, RTSP, WSS-FLV, MPEG-TS   │
│                                                                     │
│   prism_resolver.dll       URL resolution via yt-dlp               │
│   (Platform Resolver)      YouTube, Twitch, Vimeo, etc.            │
│                                                                     │
│   prism_common.lib         Shared static library (linked at build) │
│   (build-time only)        MF H.264/AAC decoders, audio ringbuf   │
├─────────────────────────────────────────────────────────────────────┤
│                       Windows APIs (built-in)                       │
│   Media Foundation  │  Direct3D 11  │  Winsock2  │  SChannel TLS   │
└─────────────────────────────────────────────────────────────────────┘
```

## Module System

See [MODULES.md](modules.md) for the full business model and module strategy.

### Module → DLL → C# Mapping

| Module (Asset Store) | Native DLL | C# Player | C# Bridge |
|---|---|---|---|
| **Prism Player (Core)** | `prism_player.dll` | `WindowsMediaEnginePlayer` | `WindowsMediaEngineBridge` |
| **Platform Resolver** | `prism_resolver.dll` | *(used by Core's PrismVideoPlayer)* | `ResolverBridge` |
| **Live Stream Pro** | `prism_live.dll` | `WindowsLivePlayer` | `LiveStreamBridge` |
| **Spatial Audio** | *(pure C#)* | *(extends any player)* | N/A |

### Auto-Detection

Modules are detected at runtime via DLL availability. No compile-time dependencies between modules.

```csharp
// PrismPlayerFactory.cs
_nativeAvailable = TryLoad(() => PlayerBridge.is_available());      // Core
_liveAvailable   = TryLoad(() => LiveStreamBridge.is_available());  // Live Stream Pro
_resolverAvailable = TryLoad(() => ResolverBridge.is_available());  // Platform Resolver
// Missing DLL → DllNotFoundException → false → module unavailable
```

### URL Routing (SelectBackend)

```
URL → Is it a live protocol?
       ├─ Yes + Live module available → WindowsLivePlayer
       ├─ Yes + Live module absent    → Error: "Live Stream Pro module required"
       └─ No  → Is it a native format? (MP4/HLS/FMP4)
                 ├─ Yes → WindowsMediaEnginePlayer
                 └─ No  → Need resolution? (YouTube/Twitch URL)
                           ├─ Yes + Resolver available → Resolve, then re-route
                           └─ No  → Unsupported format error
```

## Live Stream Plugin Architecture (prism_live.dll)

The unified live stream plugin handles all real-time protocols. Internally it dispatches by URL scheme to protocol-specific transports, which all feed into shared decode and output pipelines.

```
                       prism_live.dll
                            │
              prism_live_connect(handle, url)
                            │
                    URL scheme dispatch
                            │
         ┌──────────┬───────┴───────┬──────────┐
         │          │               │          │
    whep://    rtmp://         rtspt://    wss://*.flv
    whep+http  rtmps://        rtsp://     ws://*.flv
         │          │          rtsps://         │
         │          │               │          │
   ┌─────┴────┐ ┌──┴───┐    ┌─────┴────┐ ┌───┴────┐   ┌──────────┐
   │  WHEP    │ │ RTMP │    │  RTSP    │ │  WS    │   │  HTTP    │
   │ signaling│ │client│    │ client   │ │ client │   │ stream   │
   │ + ICE    │ │      │    │ DESCRIBE │ │ RFC6455│   │ (WinHTTP)│
   │ + DTLS   │ │      │    │ SETUP    │ │ over   │   │          │
   │ + SRTP   │ │      │    │ PLAY     │ │ TLS    │   │          │
   └────┬─────┘ └──┬───┘    └────┬─────┘ └───┬────┘   └────┬─────┘
        │          │              │           │             │
   RTP pkts   FLV tags     Interleaved    Raw FLV      Raw TS
        │          │         RTP/$-frame   bytes        bytes
        │          │              │           │             │
   ┌────┴────┐ ┌──┴───┐    ┌────┴──────┐ ┌──┴───┐    ┌───┴────┐
   │RTP H264 │ │ FLV  │    │RTP H264   │ │ FLV  │    │ MPEG-TS│
   │depack   │ │demux │    │depack     │ │stream│    │ demux  │
   │(FU-A,   │ │      │    │+ RTP AAC  │ │parser│    │PAT/PMT │
   │ STAP-A) │ │      │    │depack     │ │      │    │PES     │
   └────┬────┘ └──┬───┘    └────┬──────┘ └──┬───┘    └───┬────┘
        │         │              │           │            │
        ▼         ▼              ▼           ▼            ▼
   ┌──────────────────────────────────────────────────────────┐
   │              Shared Decode Pipeline                       │
   │                                                           │
   │   H.264 NALs (Annex B) ──→ mf_h264_decoder ──→ RGBA     │
   │                              (from prism_common)          │
   │                                                           │
   │   AAC frames ──→ mf_aac_decoder ──→ PCM float            │
   │   Opus frames ──→ opus_decoder ──→ PCM float              │
   │                                                           │
   │   avcc_to_annexb (FLV/RTMP AVCC → Annex B conversion)    │
   │   aac_adts (ADTS header synthesis for MF input)           │
   └──────────────────────────────────┬───────────────────────-┘
                                      │
   ┌──────────────────────────────────┴───────────────────────-┐
   │              Shared Output Buffers                         │
   │                                                           │
   │   RGBA frame buffer ──→ prism_live_get_frame()            │
   │   PCM ring buffer   ──→ prism_live_read_audio()           │
   │   State enum        ──→ prism_live_get_state()            │
   └───────────────────────────────────────────────────────────┘
```

### Exported C API

The live plugin exports the same API shape as the former WebRTC plugin. This means the C# player component works identically regardless of which protocol is in use.

```c
// Lifecycle
int         prism_live_init(void);
void        prism_live_shutdown(void);
void*       prism_live_create(const PrismLiveConfig* config);
void        prism_live_destroy(void* handle);

// Connection (URL determines protocol internally)
int         prism_live_connect(void* handle, const char* url);
void        prism_live_disconnect(void* handle);

// State
int         prism_live_get_state(void* handle);         // PrismLiveState enum
const char* prism_live_get_error(void* handle);
bool        prism_live_is_available(void);
const char* prism_live_get_version(void);

// Video output
void*       prism_live_get_frame(void* handle);          // Returns PrismLiveFrame*
void        prism_live_get_video_size(void* handle, int* w, int* h);

// Audio output
bool        prism_live_has_audio(void* handle);
void        prism_live_get_audio_format(void* handle, int* channels, int* sample_rate);
int         prism_live_get_audio_available(void* handle);
int         prism_live_read_audio(void* handle, float* buffer, int frames);
void        prism_live_set_audio_volume(void* handle, float volume);
```

### Supported Protocols & VRCDN Endpoints

| Protocol | URL Pattern | VRCDN Endpoint | Transport | Demux |
|---|---|---|---|---|
| **WebRTC/WHEP** | `whep+http://...` | *(not used by VRCDN)* | WHEP+ICE+DTLS | RTP H.264 + Opus |
| **RTMP** | `rtmp://stream.vrcdn.live/live/{user}` | RTMP | rtmp_client | flv_demux |
| **RTSPT** | `rtspt://stream.vrcdn.live/live/{user}` | RTSPT (PC/VR low latency) | rtsp_client | rtp_h264 + rtp_aac |
| **WSS FLV** | `wss://stream.vrcdn.live/live/{user}.flv` | FLV over WebSocket (HTML5) | ws_client | flv_stream_parser |
| **MPEG-TS** | `https://stream.vrcdn.live/live/{user}.live.ts` | MPEG-TS (Quest) | http_stream | ts_demux |
| **FMP4** | `https://stream.vrcdn.live/live/{user}.live.mp4` | FMP4 | *(Media Engine, not this plugin)* | — |

## Shared Components (prism_common.lib)

Static library linked into both `prism_live.dll` and `prism_player.dll` at build time. Not shipped as a separate DLL.

| Component | Source | Used By |
|---|---|---|
| `mf_h264_decoder` | Moved from prism-webrtc-plugin | Live plugin (all paths) |
| `mf_aac_decoder` | New | Live plugin (RTMP, RTSP, WSS-FLV, TS) |
| `audio_ringbuffer` | Moved from prism-webrtc-plugin | Live plugin (all paths) |
| `rtp_h264_depacketizer` | Moved from prism-webrtc-plugin | Live plugin (WebRTC, RTSP) |
| `rtp_opus_depacketizer` | Moved from prism-webrtc-plugin | Live plugin (WebRTC) |
| `avcc_to_annexb` | New | Live plugin (FLV/RTMP AVCC → Annex B) |
| `aac_adts` | New | Live plugin (ADTS header synthesis) |

## NativeFramePump (C# shared utility)

The `NativeFramePump` class in Core eliminates duplicated texture/audio boilerplate across player components. Every native player backend uses it.

```csharp
// Core provides this shared utility
public class NativeFramePump
{
    // Texture2D creation, resize, RGBA upload, vertical flip
    // AudioSource setup, OnAudioFilterRead, Spatialized mode
    // Surround downmix, raw sample callback
    // Pre-allocated buffers (zero GC during playback)

    public NativeFramePump(GetFrameDelegate getFrame, ReadAudioDelegate readAudio);
    public void UpdateFrame();     // Called from player's Update()
    public Texture2D Texture { get; }
    public int VideoWidth { get; }
    public int VideoHeight { get; }
}
```

Each player component becomes a thin lifecycle wrapper (~100 lines) instead of reimplementing 650+ lines of frame/audio plumbing.

## Threading Model

```
┌─────────────────────────────────────────────────────────────┐
│                     Unity Main Thread                        │
│  - PrismVideoPlayer.Update()                                │
│  - NativeFramePump.UpdateFrame() → texture upload           │
│  - OnAudioFilterRead() → audio delivery                     │
│  - Player control (play/pause/seek)                         │
└───────────────────────┬─────────────────────────────────────┘
                        │ (polls native buffers)
         ┌──────────────┴──────────────┐
         ▼                             ▼
┌─────────────────────┐    ┌──────────────────────────────┐
│  Transport Thread   │    │     Resolver Thread          │
│  (1 per connection) │    │  - yt-dlp process spawn      │
│  - Network I/O      │    │  - JSON parsing              │
│  - Demux            │    │  - URL extraction             │
│  - MF decode (sync) │    └──────────────────────────────┘
│  - Buffer write     │
└─────────────────────┘
```

MF decoder operations happen synchronously on the transport thread. The MFT is hardware-accelerated and fast. Frame buffer + ring buffer provide thread-safe boundary to Unity's main thread.

## Build System

All native components use CMake. Each repo builds independently but references `prism-common` as a subdirectory dependency.

```bash
# Build prism_live.dll (includes prism_common.lib)
cd prism-live-plugin/build
cmake -G "Visual Studio 17 2022" -A x64 ..
cmake --build . --config Release

# Build prism_player.dll
cd prism-video-player/Native/build
cmake -G "Visual Studio 17 2022" -A x64 ..
cmake --build . --config Release

# Build prism_resolver.dll
cd prism-ytdlp-plugin/build
cmake -G "Visual Studio 17 2022" -A x64 ..
cmake --build . --config Release
```

All DLLs output to `prism-unity-plugin/Plugins/Windows/x86_64/` via post-build copy.

## Platform Support

| Platform | Core Player | Live Stream | Resolver | Status |
|---|---|---|---|---|
| Windows x64 | Media Foundation | Winsock + MF | yt-dlp | Active development |
| Quest/Android | MediaCodec | *(TBD)* | yt-dlp | Planned |
| macOS | AVFoundation | *(TBD)* | yt-dlp | Planned |
| Linux | GStreamer | *(TBD)* | yt-dlp | Planned |

The C# layer (`IPrismPlayer`, `NativeFramePump`, `PrismVideoPlayer`) is platform-agnostic. Each module ships platform-specific native binaries under `Plugins/{platform}/`.

## Licensing

| Component | License |
|---|---|
| All Prism native code | Proprietary (Team Yoopi) |
| Unity C# scripts | Proprietary (Team Yoopi) |
| yt-dlp (bundled binary) | Unlicense |
| Windows APIs used | Built-in, royalty-free |

No LGPL, GPL, or copyleft dependencies. This is a key selling point vs. AVPro (LGPL via FFmpeg).

---

<div class="footer-nav">
  <a href="modules/">Module Strategy →</a> |
  <a href="building/">Building →</a>
</div>
