---
title: Home
---

# Prism Video Player

High-performance native video player for Unity with HLS/DASH streaming support.

<div class="hero">
  <p>A modular, plugin-based video player designed for professional Unity applications.</p>
</div>

## Features

- **Native Performance** - Uses OS-level media frameworks (Windows Media Foundation, AVFoundation)
- **Plugin Architecture** - Modular design separates codecs from core player
- **Streaming Support** - HLS, DASH, RTMP, RTSP protocols
- **Platform URLs** - Resolve YouTube, Twitch, Vimeo URLs to direct streams
- **Unity Integration** - Clean C# API with MonoBehaviour components

## Quick Links

| Guide | Description |
|-------|-------------|
| [Unity Integration](unity/) | Setup and usage for Unity projects |
| [FFmpeg Plugin](ffmpeg/) | Setup and usage for FFmpeg streaming support |
| [Testing Guide](testing/) | How to test streams with the native player |
| [Plugin API](api/) | Developing decoder and resolver plugins |
| [Architecture](architecture/) | System design and component overview |
| [Building](building/) | Build instructions for all platforms |

## Getting Started

### 1. Build the Native Player

```bash
cd Native
mkdir build && cd build
cmake .. -G "Visual Studio 17 2022" -A x64
cmake --build . --config Release
```

### 2. Run Stream Tests

```bash
cd Native\build\bin\Release

# Test HLS streaming
.\prism_stream_tests.exe --url "https://devstreaming-cdn.apple.com/videos/streaming/examples/img_bipbop_adv_example_fmp4/master.m3u8" --verbose
```

### 3. Integrate with Unity

Add the Unity package via Package Manager:
```
https://github.com/Team-Yoopi/prism-unity-plugin.git
```

## Repository Structure

| Repository | Description | License |
|------------|-------------|---------|
| [prism-video](https://github.com/Team-Yoopi/prism-video) | Core player + native backends | Proprietary |
| [prism-ffmpeg-plugin](https://github.com/Team-Yoopi/prism-ffmpeg-plugin) | FFmpeg decoder plugin | LGPL-2.1 |
| [prism-ytdlp-plugin](https://github.com/Team-Yoopi/prism-ytdlp-plugin) | yt-dlp URL resolver | Unlicense |
| [prism-unity-plugin](https://github.com/Team-Yoopi/prism-unity-plugin) | Unity integration | Proprietary |

## Platform Support

| Platform | Native Backend | Status |
|----------|---------------|--------|
| Windows | Media Foundation | ✅ Complete |
| macOS | AVFoundation | ⬜ Planned |
| Linux | GStreamer | ⬜ Planned |
| iOS | AVFoundation | ⬜ Planned |
| Android | MediaCodec | ⬜ Planned |

## Video Demo

<!-- Placeholder for future video demo -->
<div class="video-placeholder">
  <p><em>Video demonstration coming soon</em></p>
</div>

---

<div class="footer-nav">
  <a href="testing/">Next: Testing Guide →</a>
</div>
