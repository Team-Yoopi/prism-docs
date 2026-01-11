---
title: Unity Integration
---

# Prism Unity Plugin

High-performance native video player for Unity with automatic backend selection, HLS streaming, and platform URL resolution.

## Features

- **Unified Video Player** - Single component that auto-selects the best backend for any URL
- **Native Performance** - Uses Windows Media Engine for hardware-accelerated playback
- **Platform URL Resolution** - Play YouTube, Twitch, Vimeo URLs directly via yt-dlp
- **HLS Streaming** - Full support for live streams (Twitch, YouTube Live)
- **No FFmpeg Required** - Most content plays with native Windows APIs

## Requirements

- Unity 2020.3 or later
- 64-bit builds only
- Windows 10+ (macOS and Linux planned)

## Installation

### Via Unity Package Manager

1. Open **Window > Package Manager**
2. Click **+** > **Add package from git URL**
3. Enter:
   ```
   https://github.com/Team-Yoopi/prism-unity-plugin.git
   ```

### Manual Installation

1. Clone or download the repository
2. Copy to your project's `Packages` folder
3. Or import as a local package via Package Manager

## Quick Start

### Using PrismVideoPlayer (Recommended)

The unified `PrismVideoPlayer` component automatically handles URL resolution and backend selection:

```csharp
using Prism;
using UnityEngine;

public class SimplePlayer : MonoBehaviour
{
    public PrismVideoPlayer player;
    public Renderer targetRenderer;

    void Start()
    {
        // Just set the URL - everything else is automatic!
        player.Play("https://www.youtube.com/watch?v=dQw4w9WgXcQ");
    }
}
```

### Inspector Setup

1. Create a Quad (GameObject > 3D Object > Quad)
2. Add Component > **Prism/Video Player**
3. Set **Target Renderer** to the Quad
4. Enter any URL (YouTube, Twitch, direct MP4/HLS)
5. Enable **Play On Awake** or call `Play()` from script

## Content Type Support

### Automatic Backend Selection

The unified player automatically selects the optimal backend based on the content type:

| Content Type | Backend Used | Requirements |
|--------------|--------------|--------------|
| **Twitch Live Streams** | Windows Media Engine | Native (no plugins) |
| **YouTube Live Streams** | Windows Media Engine | Native (no plugins) |
| **YouTube VODs** | Windows Media Engine | Native (no plugins) |
| **Direct HLS (.m3u8)** | Windows Media Engine | Native (no plugins) |
| **Direct MP4/MOV/WMV** | Windows Media Engine | Native (no plugins) |
| **WebM/MKV/FLV** | FFmpeg | Requires FFmpeg plugin |
| **RTMP/RTSP Streams** | FFmpeg | Requires FFmpeg plugin |

### How It Works

```
User provides URL (YouTube, Twitch, or direct)
          │
          ▼
┌─────────────────────────┐
│   URL Resolution        │  ← yt-dlp resolves platform URLs
│   (YouTube, Twitch)     │    to direct stream URLs
└─────────────────────────┘
          │
          ▼
┌─────────────────────────┐
│   Format Detection      │  ← Checks: HLS? MP4? WebM?
│                         │
└─────────────────────────┘
          │
          ▼
┌─────────────────────────┐
│   Backend Selection     │
│                         │
│  HLS (.m3u8)           │──▶ Windows Media Engine
│  MP4/MOV/WMV           │──▶ Windows Media Engine
│  WebM/MKV/FLV          │──▶ FFmpeg (if available)
│  RTMP/RTSP             │──▶ FFmpeg (if available)
└─────────────────────────┘
          │
          ▼
┌─────────────────────────┐
│   Playback              │
│   (Hardware Accelerated)│
└─────────────────────────┘
```

### Platform URL Resolution

The yt-dlp resolver automatically handles platform URLs and requests native-compatible formats:

| Platform | Live Streams | VODs | Format Requested |
|----------|--------------|------|------------------|
| **YouTube** | HLS | MP4 (H.264) | Native compatible |
| **Twitch** | HLS | HLS | Native compatible |
| **Vimeo** | - | MP4 | Native compatible |
| **Dailymotion** | HLS | MP4 | Native compatible |
| **Facebook** | HLS | MP4 | Native compatible |
| **Twitter/X** | - | MP4 | Native compatible |
| **Instagram** | - | MP4 | Native compatible |
| **TikTok** | - | MP4 | Native compatible |

yt-dlp supports 1000+ additional platforms.

## Components

### PrismVideoPlayer

The recommended unified player component.

**Inspector Properties:**
- **URL** - Video URL (YouTube, Twitch, direct HLS/MP4)
- **Quality** - Preferred stream quality (Auto, Low, Medium, High, Full)
- **Play On Awake** - Auto-play when scene starts
- **Loop** - Loop playback
- **Volume** - Audio volume (0-1)
- **Target Renderer** - Renderer to display video on
- **Target Raw Image** - UI RawImage to display video on
- **Target Render Texture** - RenderTexture output

**Events:**
- `OnReady` - Media is prepared and ready
- `OnStarted` - Playback started
- `OnPaused` - Playback paused
- `OnStopped` - Playback stopped
- `OnFinished` - End of media reached
- `OnError` - Error occurred (string message)
- `OnStreamResolved` - Platform URL resolved (StreamInfo)

### VideoPlayerUI

Pre-built UI controller for video playback.

**Features:**
- Play/Pause button
- Progress bar with seeking
- Time display (current / total, or "LIVE" for streams)
- Volume slider
- URL input field
- Auto-hide controls

**Setup:**
1. Create UI Canvas with controls
2. Add `VideoPlayerUI` component
3. Assign `PrismVideoPlayer` reference
4. Connect UI elements (sliders, buttons, labels)

## Supported Formats

### Windows Media Engine (Native)

No additional plugins required.

| Type | Formats |
|------|---------|
| **Containers** | MP4, MOV, WMV, AVI, M4V |
| **Video Codecs** | H.264/AVC, HEVC/H.265 (with system codec) |
| **Audio Codecs** | AAC, MP3, WMA, FLAC |
| **Streaming** | HLS (.m3u8), HTTP/HTTPS |

### FFmpeg Plugin (Optional)

Required for additional formats. Download via Prism > Setup Window.

| Type | Formats |
|------|---------|
| **Containers** | MKV, WebM, FLV, TS, OGG |
| **Video Codecs** | VP8, VP9, AV1, Theora |
| **Audio Codecs** | Opus, Vorbis, AC3, DTS |
| **Streaming** | RTMP, RTSP, DASH |

## Setup

### FFmpeg Libraries (Optional)

For WebM, MKV, RTMP support, download FFmpeg via the Editor:

1. Menu: **Prism > Setup Window**
2. Click **Download FFmpeg**
3. Wait for download (~200MB)

Or run manually:
```powershell
cd Packages/prism-unity-plugin
.\Scripts\download_ffmpeg.ps1
```

### yt-dlp (Auto-Downloaded)

The yt-dlp binary is automatically downloaded on first use when resolving platform URLs. No manual setup required.

## API Reference

### PrismVideoPlayer

```csharp
// Properties
string Url { get; }
string ResolvedUrl { get; }
PrismPlayerState State { get; }
bool IsPlaying { get; }
bool IsReady { get; }
bool IsLiveStream { get; }
double Time { get; }
double Duration { get; }
float NormalizedTime { get; }
int VideoWidth { get; }
int VideoHeight { get; }
Texture VideoTexture { get; }
float Volume { get; set; }
float PlaybackSpeed { get; set; }
bool Loop { get; set; }

// Methods
void Play();
void Play(string url);
void Pause();
void Resume();
void Stop();
void Seek(double timeSeconds);
void SeekNormalized(float normalizedTime);
```

### StreamQuality Enum

```csharp
public enum StreamQuality
{
    Auto,    // Let yt-dlp choose
    Low,     // ~360p
    Medium,  // ~480p
    High,    // ~720p
    Full,    // ~1080p
    Ultra    // 4K if available
}
```

### IPrismPlayer Interface

The core player interface implemented by all player backends:

```csharp
public interface IPrismPlayer : IDisposable
{
    // Lifecycle
    void SetSource(string url);
    void Play();
    void Pause();
    void Stop();
    void Seek(double positionSeconds);

    // State
    PrismPlayerState State { get; }
    bool IsPlaying { get; }
    bool IsPrepared { get; }
    bool IsLiveStream { get; }

    // Time
    double Time { get; }
    double Duration { get; }
    float NormalizedTime { get; }

    // Video
    Texture VideoTexture { get; }
    int VideoWidth { get; }
    int VideoHeight { get; }

    // Audio
    float Volume { get; set; }

    // Playback
    float PlaybackSpeed { get; set; }
    bool Loop { get; set; }

    // Events
    event Action OnPrepared;
    event Action OnStarted;
    event Action OnPaused;
    event Action OnStopped;
    event Action OnFinished;
    event Action<string> OnError;
}
```

### PrismPlayerState

```csharp
public enum PrismPlayerState
{
    Idle,       // Initial state, no media loaded
    Opening,    // Loading/buffering media
    Playing,    // Active playback
    Paused,     // Playback paused
    Stopped,    // Playback stopped
    EndOfFile,  // Reached end of media
    Error       // Error occurred
}
```

## Advanced Usage

### YouTube/Twitch with Manual Resolution

For more control over URL resolution:

```csharp
using Prism;
using Prism.Streaming;
using UnityEngine;

public class ManualResolutionExample : MonoBehaviour
{
    public PrismVideoPlayer player;

    async void Start()
    {
        var resolver = new YtdlpResolver();

        // Resolve with specific quality
        var result = await resolver.ResolveAsync(
            "https://twitch.tv/shroud",
            StreamQuality.High
        );

        if (result.Success)
        {
            Debug.Log($"Title: {result.Title}");
            Debug.Log($"Is Live: {result.IsLiveStream}");
            Debug.Log($"Format: {result.Format}");

            // Play the resolved URL
            player.Play(result.DirectUrl);
        }
        else
        {
            Debug.LogError($"Resolution failed: {result.Error}");
        }
    }
}
```

### Handling Multiple Output Targets

```csharp
using Prism;
using UnityEngine;
using UnityEngine.UI;

public class MultiOutputExample : MonoBehaviour
{
    public PrismVideoPlayer player;
    public Renderer worldRenderer;
    public RawImage uiImage;
    public RenderTexture renderTexture;

    void Start()
    {
        // Set all output targets
        player.SetOutputTargets(
            renderTexture: renderTexture,
            renderer: worldRenderer,
            rawImage: uiImage
        );

        player.Play("https://example.com/video.mp4");
    }
}
```

### Event Handling

```csharp
using Prism;
using UnityEngine;

public class EventHandlingExample : MonoBehaviour
{
    public PrismVideoPlayer player;

    void OnEnable()
    {
        player.OnReady.AddListener(OnPlayerReady);
        player.OnStarted.AddListener(OnPlayerStarted);
        player.OnFinished.AddListener(OnPlayerFinished);
        player.OnError.AddListener(OnPlayerError);
        player.OnStreamResolved.AddListener(OnStreamResolved);
    }

    void OnDisable()
    {
        player.OnReady.RemoveListener(OnPlayerReady);
        player.OnStarted.RemoveListener(OnPlayerStarted);
        player.OnFinished.RemoveListener(OnPlayerFinished);
        player.OnError.RemoveListener(OnPlayerError);
        player.OnStreamResolved.RemoveListener(OnStreamResolved);
    }

    void OnPlayerReady() => Debug.Log("Player ready");
    void OnPlayerStarted() => Debug.Log("Playback started");
    void OnPlayerFinished() => Debug.Log("Playback finished");
    void OnPlayerError(string error) => Debug.LogError($"Error: {error}");
    void OnStreamResolved(StreamInfo info) => Debug.Log($"Resolved: {info.Title}");
}
```

## Troubleshooting

### Video plays but no audio

YouTube sometimes returns video-only streams when audio isn't available in a combined format. This is a limitation when not using FFmpeg (which can merge separate streams).

### "WebM format detected. Requires FFmpeg"

The video is in WebM format which Windows Media Engine doesn't support. Either:
- Use a different video with MP4 format
- Install FFmpeg plugin via Prism > Setup Window

### URL won't resolve

- Check internet connectivity
- Verify the URL works in a browser
- Some platforms may require login or have geo-restrictions
- yt-dlp auto-downloads on first use; check for firewall blocks

### Stream buffering or stuttering

- Check network bandwidth
- Try a lower quality setting
- Live streams require stable connections

### DllNotFoundException: prism_ffmpeg

**Cause:** FFmpeg libraries not installed.

**Solution:**
```powershell
cd Packages/prism-unity-plugin
.\Scripts\download_ffmpeg.ps1
```

Or use the Editor menu: **Prism > Setup Window > Download FFmpeg**

### Poor performance / frame drops

**Solutions:**
- Use hardware acceleration (enabled by default with Media Engine)
- Reduce video quality
- Check CPU/GPU usage
- Disable other heavy operations during playback

## Platform Support

| Platform | Native Player | FFmpeg Plugin | URL Resolver |
|----------|---------------|---------------|--------------|
| Windows x64 | Media Engine | Full | Full |
| macOS | AVFoundation (planned) | Planned | Full |
| Linux x64 | GStreamer (planned) | Planned | Full |
| iOS | - | - | - |
| Android | - | - | - |
| WebGL | - | - | - |

## License

See the LICENSE file in the package for details.

---

<div class="footer-nav">
  <a href="./">Home</a> |
  <a href="ffmpeg/">FFmpeg Plugin</a> |
  <a href="api/">API Reference</a>
</div>
