---
title: Architecture
---

# System Architecture

Prism Video uses a modular plugin-based architecture that separates concerns into distinct components.

## High-Level Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                        Unity Application                         │
├─────────────────────────────────────────────────────────────────┤
│                     prism-unity-plugin                           │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐  │
│  │ C# API      │  │ P/Invoke    │  │ MonoBehaviour Components│  │
│  │ IPrismPlayer│  │ Bridge      │  │ VideoPlayerUI           │  │
│  └─────────────┘  └─────────────┘  └─────────────────────────┘  │
├─────────────────────────────────────────────────────────────────┤
│                        Native Layer                              │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │                     prism_core.dll                          ││
│  │  ┌───────────────┐  ┌───────────────┐  ┌────────────────┐  ││
│  │  │ Plugin Loader │  │ Plugin Registry│  │ Player Core    │  ││
│  │  └───────────────┘  └───────────────┘  └────────────────┘  ││
│  │  ┌───────────────────────────────────────────────────────┐  ││
│  │  │              Native OS Backend                         │  ││
│  │  │  Windows: Media Foundation | macOS: AVFoundation      │  ││
│  │  └───────────────────────────────────────────────────────┘  ││
│  └─────────────────────────────────────────────────────────────┘│
│  ┌─────────────────────────┐  ┌─────────────────────────────┐  │
│  │   prism_ffmpeg.dll      │  │     prism_ytdlp.dll         │  │
│  │   (Decoder Plugin)      │  │     (Resolver Plugin)       │  │
│  │   LGPL-2.1              │  │     Unlicense               │  │
│  └─────────────────────────┘  └─────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

## Component Overview

### Core Player (prism-video)

The core player provides:

| Component | Purpose |
|-----------|---------|
| **Plugin Loader** | Discovers and loads plugin DLLs at runtime |
| **Plugin Registry** | Maintains lists of available decoders/resolvers |
| **Player Core** | Unified API delegating to appropriate plugins |
| **Native Backend** | OS-specific video playback (built-in) |

### Plugin Types

| Type | Interface | Purpose |
|------|-----------|---------|
| **Decoder** | `PrismDecoderFactory` | Video/audio decoding |
| **Resolver** | `PrismResolverFactory` | URL resolution |
| **Renderer** | (Future) | Custom rendering |
| **Filter** | (Future) | Video/audio effects |

## URL Resolution Flow

When opening a platform URL (YouTube, Twitch, etc.):

```
1. Application calls prism_player_open("https://twitch.tv/channel")

2. Player Core queries resolvers:
   ┌─────────────────────────────────────────────┐
   │ for each resolver in registry:              │
   │   if resolver.can_resolve(url):             │
   │     stream = resolver.resolve(url, options) │
   │     if stream.success:                      │
   │       return stream.direct_url              │
   └─────────────────────────────────────────────┘

3. yt-dlp resolver spawns process:
   yt-dlp --dump-json --no-download "https://twitch.tv/channel"

4. Resolver parses JSON, extracts direct URL:
   stream.direct_url = "https://...hls/master.m3u8"
   stream.is_live = true
   stream.headers = {"User-Agent": "..."}

5. Player Core passes direct URL to decoder:
   decoder.open(stream.direct_url, options)
```

## Decoder Selection

When opening a direct URL:

```
1. Player Core queries decoders:
   ┌───────────────────────────────────────────────┐
   │ for each decoder in registry:                 │
   │   score = decoder.can_handle(url, mime_type)  │
   │   if score > best_score:                      │
   │     best_decoder = decoder                    │
   │     best_score = score                        │
   └───────────────────────────────────────────────┘

2. Scoring factors:
   - File extension match: +20
   - MIME type match: +30
   - Protocol support (HLS, RTSP): +25
   - Plugin priority: +10 (High) / +5 (Normal)

3. Selection:
   - FFmpeg plugin: High priority, broad format support
   - Native backend: Normal priority, OS-optimized
```

## Data Flow

### Video Frame Pipeline

```
┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│   Decoder    │───►│  Video Frame │───►│    Unity     │
│   Plugin     │    │    Queue     │    │   Texture    │
└──────────────┘    └──────────────┘    └──────────────┘
                           │
                    ┌──────▼──────┐
                    │ PrismFrame  │
                    │ - data[]    │
                    │ - width     │
                    │ - height    │
                    │ - stride    │
                    │ - format    │
                    │ - pts       │
                    └─────────────┘
```

### Audio Sample Pipeline

```
┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│   Decoder    │───►│  Audio Ring  │───►│    Unity     │
│   Plugin     │    │    Buffer    │    │ AudioSource  │
└──────────────┘    └──────────────┘    └──────────────┘
                           │
                    ┌──────▼──────┐
                    │ AudioBuffer │
                    │ - samples[] │
                    │ - channels  │
                    │ - rate      │
                    │ - pts       │
                    └─────────────┘
```

## Thread Model

```
┌─────────────────────────────────────────────────────────────┐
│                        Main Thread                           │
│  - Unity callbacks                                          │
│  - Texture updates                                          │
│  - Player control (play/pause/seek)                         │
└─────────────────────────────────────────────────────────────┘
                              │
              ┌───────────────┴───────────────┐
              ▼                               ▼
┌─────────────────────────┐    ┌─────────────────────────────┐
│     Decode Thread       │    │      Resolver Thread        │
│  - Video decoding       │    │  - URL resolution           │
│  - Audio decoding       │    │  - yt-dlp process           │
│  - Frame queue mgmt     │    │  - JSON parsing             │
└─────────────────────────┘    └─────────────────────────────┘
              │
              ▼
┌─────────────────────────┐
│     Network Thread      │
│  - HTTP/HTTPS fetch     │
│  - HLS segment download │
│  - Buffer management    │
└─────────────────────────┘
```

## Memory Management

### Ownership Rules

| Resource | Owner | Lifetime |
|----------|-------|----------|
| Plugin DLL handle | Plugin Loader | Application lifetime |
| Decoder instance | Application | Until `destroy()` called |
| Video frames | Frame Queue | Until consumed by Unity |
| Resolved streams | Resolver | Until `free_stream()` called |

### Buffer Strategy

```c
// Frame queue with configurable depth
#define FRAME_QUEUE_SIZE 8

// Audio ring buffer (1 second at 48kHz stereo)
#define AUDIO_RING_SIZE (48000 * 2 * sizeof(float))
```

## Error Handling

Errors propagate through return codes:

```c
PrismError err = prism_player_open(player, url, &options);
if (err != PRISM_OK) {
    const char* msg = prism_error_string(err);
    // Handle error
}
```

| Error Code | Meaning |
|------------|---------|
| `PRISM_OK` | Success |
| `PRISM_ERROR_NOT_FOUND` | URL/file not found |
| `PRISM_ERROR_NETWORK` | Network error |
| `PRISM_ERROR_CODEC` | Unsupported codec |
| `PRISM_ERROR_TIMEOUT` | Operation timed out |

## Platform Considerations

### Windows
- Media Foundation for native playback
- DirectX 11 texture sharing with Unity
- LoadLibrary for plugin loading

### macOS (Planned)
- AVFoundation for native playback
- Metal texture sharing with Unity
- dlopen for plugin loading

### Linux (Planned)
- GStreamer for native playback
- OpenGL texture sharing with Unity
- dlopen for plugin loading

---

<div class="footer-nav">
  <a href="api/">← Plugin API</a> |
  <a href="building/">Next: Building →</a>
</div>
