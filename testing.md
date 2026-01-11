---
title: Testing Guide
---

# Stream Testing Guide

The `prism_stream_tests` application validates native player compatibility with various streaming sources and video formats.

## Prerequisites

Before running tests, you need to build the native player:

```bash
cd Native
mkdir build && cd build
cmake .. -G "Visual Studio 17 2022" -A x64
cmake --build . --config Release
```

After building, the test executable will be at:
```
Native\build\bin\Release\prism_stream_tests.exe
```

## Basic Usage

```bash
cd Native\build\bin\Release

# Show help
.\prism_stream_tests.exe --help

# List all available tests
.\prism_stream_tests.exe --list

# Run a specific test
.\prism_stream_tests.exe hls_apple --verbose
```

## Command Line Options

| Option | Description | Default |
|--------|-------------|---------|
| `--list` | List all available tests | - |
| `--all` | Run all tests | - |
| `--url <url>` | Test a specific URL directly | - |
| `--category <name>` | Run tests in a category | - |
| `--headers <hdrs>` | HTTP headers for URL | - |
| `--timeout <sec>` | Set test timeout | 30 |
| `--frames <n>` | Frames to decode per test | 100 |
| `--verbose` | Enable detailed logging | off |
| `--json` | Output results as JSON | off |

## Test Categories

### HLS Streams

HTTP Live Streaming (HLS) tests using public test streams:

```bash
# Apple's official HLS test stream
.\prism_stream_tests.exe --url "https://devstreaming-cdn.apple.com/videos/streaming/examples/img_bipbop_adv_example_fmp4/master.m3u8" --verbose

# Akamai live test stream
.\prism_stream_tests.exe --url "https://cph-p2p-msl.akamaized.net/hls/live/2000341/test/master.m3u8" --verbose

# Bitmovin Art of Motion
.\prism_stream_tests.exe --url "https://bitdash-a.akamaihd.net/content/MI201109210084_1/m3u8s/f08e80da-bf1d-4e3d-8899-f0f6155f6efa.m3u8" --verbose

# Run all HLS tests
.\prism_stream_tests.exe --category hls --verbose
```

### RTSP Streams

Real Time Streaming Protocol tests:

```bash
# Wowza public VOD test
.\prism_stream_tests.exe --url "rtsp://wowzaec2demo.streamlock.net/vod/mp4:BigBuckBunny_115k.mp4" --timeout 60 --verbose

# IP camera (replace with your camera URL)
.\prism_stream_tests.exe --url "rtsp://192.168.1.100:554/stream1" --timeout 60

# Run all RTSP tests
.\prism_stream_tests.exe --category rtsp --verbose
```

### Direct MP4/Video Files

Test direct video URLs and local files:

```bash
# Sintel trailer (direct MP4)
.\prism_stream_tests.exe --url "https://media.w3.org/2010/05/sintel/trailer_hd.mp4" --verbose

# Big Buck Bunny samples
.\prism_stream_tests.exe --url "https://sample-videos.com/video321/mp4/720/big_buck_bunny_720p_1mb.mp4"
.\prism_stream_tests.exe --url "https://sample-videos.com/video321/mp4/1080/big_buck_bunny_1080p_10mb.mp4"

# Local file
.\prism_stream_tests.exe --url "C:\Videos\sample.mp4" --verbose

# Run all local file tests
.\prism_stream_tests.exe --category local --verbose
```

### RTMP Streams

RTMP requires a running RTMP server:

```bash
# Local RTMP server
.\prism_stream_tests.exe --url "rtmp://localhost/live/stream" --verbose

# Run all RTMP tests
.\prism_stream_tests.exe --category rtmp --verbose
```

## Testing Platform URLs

**Note:** YouTube, Twitch, and Vimeo URLs require the yt-dlp resolver plugin to convert them to direct URLs. The native player tests are for direct stream URLs that Windows Media Foundation can handle.

For platform URL testing, you need to:

1. Use yt-dlp to resolve the URL first
2. Pass the resolved direct URL to the test

Example workflow:
```bash
# Resolve with yt-dlp
yt-dlp -g "https://www.youtube.com/watch?v=dQw4w9WgXcQ"

# Test the resolved URL
.\prism_stream_tests.exe --url "https://resolved-direct-url.com/video.mp4" --verbose
```

## Understanding Test Output

### Successful Test

```
Testing: hls_apple
  [PASS] hls_apple (100 frames, 245.3ms to first frame, 1920x1080, HW accel)
```

Key metrics:
- **Frames decoded**: Number of video frames successfully decoded
- **Time to first frame**: Latency from open to first frame
- **Resolution**: Detected video dimensions
- **HW accel**: Hardware acceleration status

### Failed Test

```
Testing: rtsp_live
  [FAIL] rtsp_live
    Error: Open failed: Network connection timed out
```

Common failure reasons:
- Network timeout
- Unsupported codec
- Invalid URL
- Server unavailable

### Timeout

```
Testing: slow_stream
  [TIMEOUT] slow_stream
    Error: Timeout after 30.0 seconds (decoded 15 frames)
```

Increase timeout for slow streams:
```bash
.\prism_stream_tests.exe --url "..." --timeout 120
```

## JSON Output for CI

Generate JSON output for automated testing:

```bash
.\prism_stream_tests.exe --all --json > results.json
```

Example JSON output:
```json
{
  "platform": "Media Foundation",
  "version": "1.0.0",
  "tests": [
    {
      "name": "hls_apple",
      "result": "PASS",
      "time_to_open_ms": 156.32,
      "time_to_first_frame_ms": 245.31,
      "total_time_ms": 3421.15,
      "frames_decoded": 100,
      "width": 1920,
      "height": 1080,
      "fps": 30.00,
      "is_live": false,
      "is_hw_accelerated": true,
      "error": ""
    }
  ],
  "summary": {
    "total": 1,
    "passed": 1,
    "failed": 0,
    "skipped": 0,
    "timeout": 0,
    "total_time_ms": 3421.15
  }
}
```

## Custom Test URLs

Edit `Native/test/test_urls.h` to customize test URLs:

```c
// Override before including
#define PRISM_TEST_TWITCH_LIVE "https://www.twitch.tv/your_channel"
#define PRISM_TEST_YOUTUBE_LIVE_1 "https://www.youtube.com/watch?v=your_video"

#include "test_urls.h"
```

Rebuild after changes:
```bash
cmake --build . --config Release
```

## Troubleshooting

### "Failed to initialize native player"

Windows Media Foundation is not available. Ensure:
- Running on Windows 7 or later
- Media Foundation components are installed

### "Open failed: Codec not supported"

The video codec is not supported by Windows Media Foundation. Consider:
- Using the FFmpeg plugin for broader codec support
- Converting the video to H.264/AAC

### Slow first frame time

Normal for HLS/DASH streams due to:
- Manifest download
- Segment buffering
- Adaptive bitrate selection

### Network timeouts

Increase timeout for slow connections:
```bash
.\prism_stream_tests.exe --url "..." --timeout 120
```

## Video Demo

<!-- Placeholder for video demonstration -->
<div class="video-placeholder" style="background: #1a1a2e; padding: 40px; text-align: center; border-radius: 8px; margin: 20px 0;">
  <p style="color: #888; font-style: italic;">Video demonstration coming soon</p>
  <p style="color: #666; font-size: 0.9em;">This section will contain a video walkthrough of the testing process</p>
</div>

---

<div class="footer-nav">
  <a href="./">← Home</a> |
  <a href="api/">Next: Plugin API →</a>
</div>
