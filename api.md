---
title: Plugin API
---

# Plugin API Reference

Prism Video uses a plugin-based architecture where decoders and URL resolvers are loaded dynamically at runtime.

## Plugin Types

| Type | Purpose | Example |
|------|---------|---------|
| `PRISM_PLUGIN_TYPE_DECODER` | Video/audio decoding | FFmpeg, native backends |
| `PRISM_PLUGIN_TYPE_RESOLVER` | URL resolution | yt-dlp |
| `PRISM_PLUGIN_TYPE_RENDERER` | Video rendering | (future) |
| `PRISM_PLUGIN_TYPE_FILTER` | Video/audio filters | (future) |

## Required Plugin Exports

Every plugin must export these four functions:

```c
// Get plugin information
PRISM_PLUGIN_EXPORT const PrismPluginInfo* PRISM_CALL
prism_plugin_get_info(void);

// Initialize plugin (called once at load)
PRISM_PLUGIN_EXPORT PrismError PRISM_CALL
prism_plugin_init(const char* config);

// Shutdown plugin (called before unload)
PRISM_PLUGIN_EXPORT void PRISM_CALL
prism_plugin_shutdown(void);

// Register with the plugin system
PRISM_PLUGIN_EXPORT PrismError PRISM_CALL
prism_plugin_register(PrismPluginRegistry* registry);
```

## Plugin Info Structure

```c
typedef struct PrismPluginInfo {
    uint32_t api_version;           // PRISM_PLUGIN_API_VERSION
    PrismPluginType type;           // Plugin type
    const char* name;               // Display name
    const char* identifier;         // Unique ID (e.g., "com.prism.ffmpeg")
    const char* version;            // Version string
    const char* description;        // Description
    const char* license;            // License (e.g., "LGPL-2.1")
    const char* author;             // Author name
    const char* url;                // Project URL
    PrismPluginPriority priority;   // Selection priority
    uint32_t capabilities;          // Plugin-specific capabilities
} PrismPluginInfo;
```

## Decoder Plugin Interface

### Decoder Factory

```c
typedef struct PrismDecoderFactory {
    // Create a new decoder instance
    PrismDecoder* (*create)(void);

    // Destroy a decoder instance
    void (*destroy)(PrismDecoder* decoder);

    // Get decoder capabilities
    const PrismDecoderInfo* (*get_info)(void);

    // Check if decoder can handle URL/mime type (returns score 0-100)
    int (*can_handle)(const char* url, const char* mime_type);
} PrismDecoderFactory;
```

### Decoder VTable

```c
typedef struct PrismDecoderVTable {
    // Open media
    PrismError (*open)(PrismDecoder* decoder, const char* url,
                       const PrismDecoderOptions* options);

    // Playback control
    PrismError (*play)(PrismDecoder* decoder);
    PrismError (*pause)(PrismDecoder* decoder);
    PrismError (*stop)(PrismDecoder* decoder);
    PrismError (*seek)(PrismDecoder* decoder, double position_seconds);

    // State
    PrismState (*get_state)(PrismDecoder* decoder);
    double (*get_position)(PrismDecoder* decoder);
    double (*get_duration)(PrismDecoder* decoder);

    // Frame access
    PrismError (*get_video_frame)(PrismDecoder* decoder,
                                   PrismVideoFrame* frame);
    PrismError (*get_audio_samples)(PrismDecoder* decoder,
                                     PrismAudioBuffer* buffer);

    // Settings
    void (*set_loop)(PrismDecoder* decoder, bool loop);
    void (*set_speed)(PrismDecoder* decoder, float speed);
    void (*set_volume)(PrismDecoder* decoder, float volume);
} PrismDecoderVTable;
```

### Example Decoder Plugin

```c
// ffmpeg_plugin.c
#include <prism/prism_plugin.h>
#include <prism/prism_decoder.h>

static const PrismPluginInfo s_plugin_info = {
    .api_version = PRISM_PLUGIN_API_VERSION,
    .type = PRISM_PLUGIN_TYPE_DECODER,
    .name = "FFmpeg Decoder",
    .identifier = "com.prism.ffmpeg",
    .version = "1.0.0",
    .license = "LGPL-2.1",
    .priority = PRISM_PLUGIN_PRIORITY_HIGH
};

PRISM_PLUGIN_EXPORT const PrismPluginInfo* PRISM_CALL
prism_plugin_get_info(void) {
    return &s_plugin_info;
}

PRISM_PLUGIN_EXPORT PrismError PRISM_CALL
prism_plugin_init(const char* config) {
    // Initialize FFmpeg
    av_register_all();
    avformat_network_init();
    return PRISM_OK;
}

PRISM_PLUGIN_EXPORT void PRISM_CALL
prism_plugin_shutdown(void) {
    avformat_network_deinit();
}

PRISM_PLUGIN_EXPORT PrismError PRISM_CALL
prism_plugin_register(PrismPluginRegistry* registry) {
    return prism_register_decoder(registry,
        "com.prism.ffmpeg", &g_ffmpeg_decoder_factory);
}
```

## Resolver Plugin Interface

### Resolver Factory

```c
typedef struct PrismResolverFactory {
    // Create a new resolver instance
    PrismResolver* (*create)(void);

    // Destroy a resolver instance
    void (*destroy)(PrismResolver* resolver);

    // Get resolver information
    const PrismResolverInfo* (*get_info)(void);

    // Check if resolver can handle URL
    bool (*can_handle)(const char* url);
} PrismResolverFactory;
```

### Resolver VTable

```c
typedef struct PrismResolverVTable {
    // Check if URL can be resolved
    bool (*can_resolve)(PrismResolver* resolver, const char* url);

    // Resolve URL synchronously
    PrismResolvedStream* (*resolve)(PrismResolver* resolver,
                                     const char* url,
                                     const PrismResolverOptions* options);

    // Resolve URL asynchronously
    PrismError (*resolve_async)(PrismResolver* resolver,
                                 const char* url,
                                 const PrismResolverOptions* options,
                                 PrismResolverCallback callback,
                                 void* user_data);

    // Cancel async resolution
    void (*cancel)(PrismResolver* resolver);

    // Free resolved stream
    void (*free_stream)(PrismResolvedStream* stream);

    // Check if resolver tool is available
    bool (*is_available)(PrismResolver* resolver);

    // Ensure resolver tool is available (download if needed)
    PrismError (*ensure_available)(PrismResolver* resolver,
                                    void (*progress)(float, void*),
                                    void* user_data);
} PrismResolverVTable;
```

### Resolved Stream Structure

```c
typedef struct PrismResolvedStream {
    char* direct_url;           // Direct playback URL
    char* audio_url;            // Separate audio URL (if any)
    char* title;                // Stream/video title
    char* channel;              // Channel/author name
    int width, height;          // Video dimensions
    double duration;            // Duration in seconds
    bool is_live;               // Live stream flag
    bool is_hls;                // HLS stream flag
    bool success;               // Resolution success
    char* error;                // Error message (if failed)

    // HTTP headers for playback
    char** header_names;
    char** header_values;
    int header_count;

    // Expiration (for time-limited URLs)
    int64_t expires_at;         // Unix timestamp
} PrismResolvedStream;
```

## Plugin Discovery

Plugins are discovered by scanning directories for files matching:

| Platform | Pattern |
|----------|---------|
| Windows | `prism_*.dll` |
| macOS | `libprism_*.dylib` |
| Linux | `libprism_*.so` |

Default search paths:
- Current directory
- `./plugins/`
- `../plugins/`
- System paths (Linux: `/usr/lib/prism/plugins`)

## Error Codes

```c
typedef enum PrismError {
    PRISM_OK = 0,
    PRISM_ERROR_INVALID_PARAMETER = -1,
    PRISM_ERROR_OUT_OF_MEMORY = -2,
    PRISM_ERROR_NOT_SUPPORTED = -3,
    PRISM_ERROR_NOT_FOUND = -4,
    PRISM_ERROR_IO = -5,
    PRISM_ERROR_NETWORK = -6,
    PRISM_ERROR_CODEC = -7,
    PRISM_ERROR_FORMAT = -8,
    PRISM_ERROR_TIMEOUT = -9,
    PRISM_ERROR_CANCELLED = -10,
    PRISM_ERROR_ALREADY_INITIALIZED = -11,
    PRISM_ERROR_NOT_INITIALIZED = -12
} PrismError;
```

---

<div class="footer-nav">
  <a href="testing/">← Testing Guide</a> |
  <a href="architecture/">Next: Architecture →</a>
</div>
