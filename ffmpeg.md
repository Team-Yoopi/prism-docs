---
title: FFmpeg Plugin
---

# Prism FFmpeg Plugin

Optional FFmpeg-based decoder plugin for Prism Video Player, providing broad codec and streaming protocol support.

## Overview

The FFmpeg plugin extends Prism with support for:

- **Streaming Protocols**: HLS (.m3u8), DASH (.mpd), RTMP, RTSP
- **Video Codecs**: H.264, HEVC/H.265, VP8, VP9, AV1, ProRes, DNxHD
- **Audio Codecs**: AAC, MP3, Opus, Vorbis, FLAC, AC3, DTS
- **Container Formats**: MKV, WebM, FLV, TS, OGV

## Do I Need This Plugin?

**Without FFmpeg Plugin:**
- Windows Media Foundation handles MP4, MOV, WMV, MP3 via direct HTTP/HTTPS URLs
- Works with most direct video file links

**With FFmpeg Plugin:**
- HLS streaming (live and VOD)
- RTMP/RTSP protocols
- Additional codecs (VP9, AV1, HEVC)
- More format compatibility

## Installation

### Prerequisites

- CMake 3.16+
- Visual Studio 2019+ (Windows)
- FFmpeg 5.0+ (bundled or system)

### Option 1: Use Pre-built Binaries

Download the latest release from the [Releases page](https://github.com/Team-Yoopi/prism-ffmpeg-plugin/releases) and copy the DLLs to your Unity project's `Plugins/Windows/x86_64` folder.

### Option 2: Build from Source

```bash
# Clone the repository
git clone https://github.com/Team-Yoopi/prism-ffmpeg-plugin.git
cd prism-ffmpeg-plugin

# Download FFmpeg (or use your own)
mkdir -p ffmpeg/windows
# Download from https://github.com/BtbN/FFmpeg-Builds/releases
# Extract include/, lib/, and bin/ to ffmpeg/windows/

# Configure and build
mkdir build && cd build
cmake .. -G "Visual Studio 17 2022" -A x64 -DPRISM_CORE_DIR="../../prism-video/Native"
cmake --build . --config Release
```

### Using System FFmpeg (Linux/macOS)

```bash
cmake .. -DPRISM_USE_SYSTEM_FFMPEG=ON
cmake --build .
```

## Configuration

### CMake Options

| Option | Default | Description |
|--------|---------|-------------|
| `BUILD_SHARED_LIBS` | ON | Build as shared library (DLL) |
| `PRISM_USE_SYSTEM_FFMPEG` | OFF | Use system FFmpeg instead of bundled |
| `PRISM_CORE_DIR` | `../prism-video/Native` | Path to Prism core headers |
| `BUILD_TESTS` | ON | Build test executable |

### FFmpeg Directory Structure

When using bundled FFmpeg, organize the directory as:

```
ffmpeg/
└── windows/
    ├── include/
    │   ├── libavcodec/
    │   ├── libavformat/
    │   ├── libavutil/
    │   ├── libswscale/
    │   └── libswresample/
    ├── lib/
    │   ├── avcodec.lib
    │   ├── avformat.lib
    │   └── ...
    └── bin/
        ├── avcodec-60.dll
        ├── avformat-60.dll
        └── ...
```

## Testing

### Run the Test Suite

```bash
cd build/bin/Release

# List all available tests
prism_ffmpeg_tests.exe --list

# Run all HLS tests
prism_ffmpeg_tests.exe --category hls --verbose

# Test a specific HLS stream
prism_ffmpeg_tests.exe hls_apple

# Test a custom URL
prism_ffmpeg_tests.exe --url "https://your-stream.m3u8" --frames 200

# JSON output for CI
prism_ffmpeg_tests.exe --all --json
```

### Test Options

| Option | Description |
|--------|-------------|
| `--list` | List all available tests |
| `--all` | Run all tests |
| `--category <name>` | Run tests in category: hls, rtsp, rtmp, codec |
| `--url <url>` | Test a specific URL directly |
| `--timeout <sec>` | Set test timeout (default: 60) |
| `--frames <n>` | Frames to decode per test (default: 100) |
| `--verbose` | Enable verbose logging |
| `--json` | Output results as JSON |

### Test Categories

| Category | Tests |
|----------|-------|
| **hls** | Apple HLS, Akamai Live, Bitmovin, Mux |
| **rtsp** | Wowza VOD |
| **rtmp** | Local server (requires setup) |
| **codec** | MP4 H.264 baseline tests |

## API Usage

### Direct Factory Access

```c
#include "prism_ffmpeg_plugin.h"

// Get the FFmpeg decoder factory
const PrismDecoderFactory* factory = prism_ffmpeg_get_factory();

// Create a decoder instance
PrismDecoder* decoder = factory->create();

// Set options
PrismDecoderOpenOptions options;
prism_decoder_options_init(&options);
options.timeout_ms = 30000;
options.enable_video = true;
options.enable_audio = true;

// Open media
PrismError err = decoder->vtable->open(decoder, "https://example.com/stream.m3u8", &options);
if (err != PRISM_OK) {
    printf("Error: %s\n", decoder->vtable->get_error_message(decoder));
}

// Start playback
decoder->vtable->play(decoder);

// Update loop
while (running) {
    int frames = decoder->vtable->update(decoder, 0.016);
    if (frames > 0) {
        const PrismVideoFrame* frame = decoder->vtable->get_video_frame(decoder);
        // Render frame...
    }
}

// Cleanup
decoder->vtable->stop(decoder);
decoder->vtable->close(decoder);
decoder->vtable->destroy(decoder);
```

### Query FFmpeg Version

```c
printf("FFmpeg version: %s\n", prism_ffmpeg_get_version());
```

## Supported Formats

### Streaming Protocols

| Protocol | Support |
|----------|---------|
| HLS (HTTP Live Streaming) | Full |
| DASH (Dynamic Adaptive Streaming) | Full |
| RTMP (Real-Time Messaging Protocol) | Full |
| RTSP (Real-Time Streaming Protocol) | Full |
| HTTP/HTTPS | Full |
| File (local) | Full |

### Video Codecs

| Codec | Hardware Accel |
|-------|----------------|
| H.264/AVC | DXVA2, NVDEC |
| H.265/HEVC | DXVA2, NVDEC |
| VP8 | Software |
| VP9 | DXVA2, NVDEC |
| AV1 | NVDEC |
| MPEG-2 | DXVA2 |
| MPEG-4 | DXVA2 |

### Audio Codecs

- AAC (LC, HE-AAC, HE-AACv2)
- MP3
- Opus
- Vorbis
- FLAC
- AC3/E-AC3
- DTS

## Troubleshooting

### "FFmpeg not found" Error

Ensure FFmpeg libraries are in the correct location:
- Windows: `ffmpeg/windows/lib/*.lib` and `ffmpeg/windows/bin/*.dll`
- Or set `PRISM_USE_SYSTEM_FFMPEG=ON`

### HLS Stream Won't Open

- Check network connectivity
- Verify the URL is accessible (try in a browser)
- Increase timeout: `--timeout 120`
- Check for required headers (some streams require User-Agent)

### No Video Output

- Verify the stream has a video track
- Check decoder state: `decoder->vtable->get_state(decoder)`
- Enable verbose logging for debugging

## License

LGPL-2.1 due to FFmpeg dependency. The plugin can be dynamically linked to your application.

---

<div class="footer-nav">
  <a href="./">← Home</a> |
  <a href="api/">API Reference →</a>
</div>
