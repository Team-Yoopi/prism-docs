---
title: Building
---

# Build Instructions

Complete build instructions for all platforms and components.

## Prerequisites

### All Platforms

- CMake 3.16 or later
- Git

### Windows

- Visual Studio 2019 or 2022 with C++ workload
- Windows SDK 10.0.18362.0 or later

### macOS (Planned)

- Xcode 12 or later
- macOS 10.15 SDK or later

### Linux (Planned)

- GCC 9+ or Clang 10+
- GStreamer 1.16+ development packages

## Building the Core Player

### Windows

```bash
# Clone the repository
git clone https://github.com/Team-Yoopi/prism-video.git
cd prism-video/Native

# Create build directory
mkdir build && cd build

# Generate Visual Studio solution
cmake .. -G "Visual Studio 17 2022" -A x64

# Build Release configuration
cmake --build . --config Release

# Build Debug configuration (optional)
cmake --build . --config Debug
```

Output files:
```
Native/build/bin/Release/
├── prism_core.dll          # Core player library
├── prism_stream_tests.exe  # Test application
└── prism_core.lib          # Import library
```

### CMake Options

| Option | Default | Description |
|--------|---------|-------------|
| `PRISM_BUILD_TESTS` | ON | Build test applications |
| `PRISM_BUILD_SHARED` | ON | Build shared library |
| `CMAKE_BUILD_TYPE` | Release | Build type (Debug/Release) |

## Building Plugins

### FFmpeg Plugin

```bash
# Clone the plugin repository
git clone https://github.com/Team-Yoopi/prism-ffmpeg-plugin.git
cd prism-ffmpeg-plugin

# Create build directory
mkdir build && cd build

# Configure with system FFmpeg
cmake .. -G "Visual Studio 17 2022" -A x64 \
    -DPRISM_VIDEO_DIR=../../prism-video/Native

# Or use bundled FFmpeg
cmake .. -G "Visual Studio 17 2022" -A x64 \
    -DPRISM_VIDEO_DIR=../../prism-video/Native \
    -DFFMPEG_USE_BUNDLED=ON

# Build
cmake --build . --config Release
```

#### FFmpeg CMake Options

| Option | Default | Description |
|--------|---------|-------------|
| `PRISM_VIDEO_DIR` | Required | Path to prism-video/Native |
| `FFMPEG_USE_BUNDLED` | OFF | Download and build FFmpeg |
| `FFMPEG_ROOT` | Auto | Path to FFmpeg installation |

### yt-dlp Plugin

```bash
# Clone the plugin repository
git clone https://github.com/Team-Yoopi/prism-ytdlp-plugin.git
cd prism-ytdlp-plugin

# Create build directory
mkdir build && cd build

# Configure
cmake .. -G "Visual Studio 17 2022" -A x64 \
    -DPRISM_VIDEO_DIR=../../prism-video/Native

# Build
cmake --build . --config Release
```

## Installing Plugins

After building, copy plugin DLLs to one of these locations:

```
# Same directory as prism_core.dll
Native/build/bin/Release/
├── prism_core.dll
├── prism_ffmpeg.dll    # Copy here
└── prism_ytdlp.dll     # Copy here

# Or a plugins subdirectory
Native/build/bin/Release/
├── prism_core.dll
└── plugins/
    ├── prism_ffmpeg.dll
    └── prism_ytdlp.dll
```

## Unity Integration

### Install via Package Manager

1. Open Unity Package Manager (Window → Package Manager)
2. Click the + button → "Add package from git URL"
3. Enter: `https://github.com/Team-Yoopi/prism-unity-plugin.git`
4. Click Add

### Manual Installation

1. Clone the Unity plugin:
   ```bash
   git clone https://github.com/Team-Yoopi/prism-unity-plugin.git
   ```

2. Copy the contents to your Unity project's Packages folder

3. Copy native binaries to `Plugins/x86_64/`:
   - `prism_core.dll`
   - `prism_ffmpeg.dll` (optional)
   - `prism_ytdlp.dll` (optional)

## Development Workflow

### Recommended Directory Structure

```
workspace/
├── prism-video/           # Core player (private)
├── prism-ffmpeg-plugin/   # FFmpeg plugin (LGPL)
├── prism-ytdlp-plugin/    # yt-dlp plugin (Unlicense)
└── prism-unity-plugin/    # Unity integration (private)
```

### Quick Rebuild Script (Windows)

Create `rebuild.bat`:
```batch
@echo off
cd prism-video\Native\build
cmake --build . --config Release

cd ..\..\..\prism-ffmpeg-plugin\build
cmake --build . --config Release

cd ..\..\..\prism-ytdlp-plugin\build
cmake --build . --config Release

echo Build complete!
```

### Copying Plugins to Unity

Create `deploy.bat`:
```batch
@echo off
set UNITY_PLUGINS=path\to\UnityProject\Assets\Plugins\x86_64

copy prism-video\Native\build\bin\Release\prism_core.dll %UNITY_PLUGINS%
copy prism-ffmpeg-plugin\build\Release\prism_ffmpeg.dll %UNITY_PLUGINS%
copy prism-ytdlp-plugin\build\Release\prism_ytdlp.dll %UNITY_PLUGINS%

echo Deployed to Unity!
```

## Troubleshooting

### "CMake Error: Could not find Visual Studio"

Ensure Visual Studio is installed with the "Desktop development with C++" workload.

### "LNK2019: unresolved external symbol"

Check that:
- All dependencies are built for the same architecture (x64)
- Import libraries (.lib) are linked correctly
- Runtime libraries match (MD vs MT)

### "Plugin not found" at runtime

Verify:
- Plugin DLL is in correct location (same dir as exe or ./plugins/)
- Plugin filename matches pattern (`prism_*.dll`)
- All plugin dependencies are present (e.g., FFmpeg DLLs)

### FFmpeg DLLs missing

When using system FFmpeg, ensure these DLLs are in PATH or copied to output:
- avcodec-XX.dll
- avformat-XX.dll
- avutil-XX.dll
- swresample-XX.dll
- swscale-XX.dll

---

<div class="footer-nav">
  <a href="architecture/">← Architecture</a> |
  <a href="./">Home</a>
</div>
