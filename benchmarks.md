---
title: Benchmarks
---

# Benchmark Comparison

Transparent, reproducible performance benchmarks comparing Prism against every major Unity video player plugin. Raw data published. Methodology open. No cherry-picking.

## Players Tested

| Player | Price | License | FFmpeg Required |
|--------|-------|---------|-----------------|
| **Prism Video Player** | $15-85 (modular) | Commercial (per-seat) | No |
| AVPro Video Ultra | $900 | Commercial (perpetual) | No |
| VLC for Unity | $700/yr | Commercial (subscription) | Yes (bundled) |
| HISPlayer | ~414 EUR/yr | Commercial | No |
| Unity VideoPlayer | Free | Built-in | No |

## Test Scenarios

| ID | Type | Source | Resolution | Codec |
|----|------|--------|------------|-------|
| `vod-local-1080p` | Local file | Big Buck Bunny (CC-BY) | 1920x1080 | H.264+AAC |
| `vod-local-4k` | Local file | Big Buck Bunny 4K | 3840x2160 | H.264+AAC |
| `vod-http-1080p` | HTTP progressive | Self-hosted CDN | 1920x1080 | H.264 |
| `vod-hls-adaptive` | HLS | Apple public test streams | 360p-1080p | H.264 |
| `live-rtmp` | RTMP | OBS to VRCDN/local | 1080p | H.264+AAC |
| `live-fmp4` | FMP4/HTTP | VRCDN endpoint | 1080p | H.264+AAC |
| `live-wss-flv` | WSS-FLV | VRCDN endpoint | 1080p | H.264+AAC |

### Player Capability Matrix

Not every player supports every scenario. Unsupported combinations are shown as "Not Supported" (gray) on the results page, never as zero.

| Scenario | Prism | AVPro | VLC | HISPlayer | Unity VP |
|----------|-------|-------|-----|-----------|----------|
| Local files | Yes | Yes | Yes | Yes | Yes |
| HTTP progressive | Yes | Yes | Yes | Yes | Yes |
| HLS | Yes | Yes | Yes | Yes | **No** |
| RTMP | Yes | Yes | Yes | No | **No** |
| WSS-FLV | Yes | No | No | No | **No** |

---

## Methodology

### Metrics

| Metric | How Measured | Notes |
|--------|-------------|-------|
| **Time to First Frame (TTFF)** | `Stopwatch` from `Open()` to first non-black pixel via `AsyncGPUReadback` of center pixel | Black threshold: 16/255. Coarse fallback: `IsPlaying && VideoWidth > 0`. Published numbers use the precise (readback) method. |
| **Frame Drops** | Compare expected frames (`elapsed x videoFPS`) vs observed texture changes per `Update()` | For AVPro: uses native `GetDroppedFrameCount()`. For others: texture pointer change tracking. |
| **Seek Latency** | `Stopwatch` from `Seek()` to `CurrentTime` within 0.1s of target | Tested at 25%, 50%, 75% of duration. 5 seeks per position. VOD only. |
| **CPU Usage** | `Process.TotalProcessorTime` sampled every 100ms, reported as % delta from baseline | Baseline = no player active. Run in built player, not Editor. |
| **Memory (RAM)** | `Process.WorkingSet64` delta from baseline, sampled every 500ms | Reports average and peak delta. |
| **Memory (VRAM)** | `Profiler.GetAllocatedMemoryForGraphicsDriver()` delta from baseline | Also reports expected VRAM from video texture dimensions. |
| **Binary Size** | Scan plugin directory at startup per player | Total DLL/bundle size. Unity VideoPlayer is built-in (0 MB additional). |
| **Rebuffering** | Track `CurrentTime` freeze >500ms while `IsPlaying` is true | Live stream scenarios only. Counts events and total stall duration. |

### Statistical Rigor

- **10 measured runs** per (scenario, player) combination
- **First run is warmup** (excluded from published statistics)
- **GC.Collect** + **2 second cooldown** between every run
- Reports: **median** (primary), mean, standard deviation, P5, P95, min, max
- All raw data is published alongside aggregated results

### Fairness Measures

These are non-negotiable for the comparison to be credible:

- **Full licenses purchased** for all commercial players (no trial/watermarked builds, which add rendering overhead)
- **Default/recommended settings** per player (not intentionally configured to disadvantage competitors)
- **Competitor wins shown clearly** — where AVPro or VLC beats Prism on a metric, that's displayed without qualification
- **"Not Supported" is honest** — shown as gray, not as zero or failure
- **Version numbers on everything** — results may change with plugin updates
- **Built standalone player** — never Unity Editor, which adds unpredictable overhead
- **Clean system** — no unnecessary background apps, antivirus state documented
- **Raw data published** — anyone can download, verify, and re-analyze

---

## Running the Benchmarks

The benchmark suite is a standalone Unity project at [`prism-benchmark/`](https://github.com/Team-Yoopi/prism-benchmark).

### Prerequisites

- Unity 6 (6000.x)
- Windows 10/11 x64
- Prism Video Player package (included via local UPM reference)
- (Optional) Competitor plugin licenses for full comparison

### Setup

1. **Clone the benchmark project**

   ```
   git clone https://github.com/Team-Yoopi/prism-benchmark.git
   ```

2. **Open in Unity 6**

   Open the `prism-benchmark/` folder as a Unity project. Prism is referenced as a local UPM package from the sibling `prism-unity-plugin/` directory.

3. **Download test content**

   Run the download script to get Big Buck Bunny test files (CC-BY licensed):

   ```powershell
   .\download-test-content.ps1
   ```

   This places MP4 files into `Assets/TestContent/`.

4. **Import competitor plugins** (optional)

   If you own licenses for AVPro Video, VLC for Unity, or HISPlayer, import them into the project. The benchmark suite auto-detects available plugins via assembly version defines:

   | Plugin | Define Symbol (auto-set) |
   |--------|--------------------------|
   | Prism | `PRISM_AVAILABLE` |
   | AVPro Video | `AVPRO_AVAILABLE` |
   | VLC for Unity | `VLC_AVAILABLE` |
   | HISPlayer | `HISPLAYER_AVAILABLE` |

5. **Create a BenchmarkConfig asset**

   In the Unity Editor: **Assets > Create > Prism Benchmark > Config**

   Configure settings:
   - `Runs Per Test`: 10 (default)
   - `Playback Duration`: 30 seconds
   - `Cooldown Seconds`: 2
   - `Output Directory`: Results

6. **Set up the scene**

   Open `Assets/Scenes/BenchmarkMain.unity` (or create a new scene). Add an empty GameObject with the `BenchmarkBootstrap` component and assign your BenchmarkConfig asset to it.

### Running

**Important:** Always run benchmarks from a **built standalone player**, not the Unity Editor. Editor overhead (Inspector updates, Scene view rendering, domain reloads) will skew every metric.

1. **Build the player**

   File > Build Settings > select your scene > Build (Windows x64).

2. **Run with the UI**

   Launch the built executable. The benchmark UI panel appears on the left side of the screen:
   - Toggle players on/off
   - Toggle scenarios on/off
   - Click **START BENCHMARK**
   - Press **F1** to show/hide the panel

3. **Run headless (auto-run mode)**

   ```
   PrismBenchmark.exe --autorun
   ```

   Runs all available players against all applicable scenarios, saves results, and quits automatically.

### Output

Results are saved to the `Results/` directory:

| File | Contents |
|------|----------|
| `results_<id>.json` | Complete raw + aggregated data with all metrics |
| `results.json` | Website-friendly format (copy to `Website/data/` for the comparison page) |
| `summary_<id>.csv` | Aggregated: median, mean, stddev, P5, P95 per metric per combination |
| `raw_<id>.csv` | Every individual run with all metric values |
| `benchmark_log_<timestamp>.txt` | Full timestamped log |

### Publishing Results to the Website

1. Copy `Results/results.json` to `Website/data/results.json`
2. Optionally copy the CSV files to `Website/data/` for download links
3. Open `Website/index.html` in a browser (or deploy to any static host)

The website automatically loads `data/results.json` and renders Chart.js visualizations for all metrics.

---

## Adding a New Player Adapter

To benchmark a player not yet included:

1. Create `Assets/Benchmark/Adapters/YourPlayerAdapter.cs` implementing `IPlayerAdapter` (or extending `PlayerAdapterBase`)
2. Add a new entry to the `PlayerType` enum in `PlayerAdapterFactory.cs`
3. Add the factory case, guarded by a `#if YOUR_DEFINE` preprocessor directive
4. Add a version define entry in `PrismBenchmark.asmdef` so the define is auto-set when the package is present

The adapter must implement:
- **Open/Play/Pause/Stop/Seek/Close** — map to the player's API
- **IsPlaying/IsPrepared/CurrentTime/Duration/VideoWidth/VideoHeight** — state polling
- **OutputTexture** — the `Texture` the player renders to (used for TTFF pixel detection and frame tracking)
- **GetPluginBinarySize()** — return total DLL size on disk

---

## Architecture

```
BenchmarkBootstrap
  └─ BenchmarkRunner (coroutine orchestrator)
       ├─ For each scenario:
       │    ├─ For each player:
       │    │    ├─ Skip if !adapter.SupportsScenario
       │    │    ├─ Warmup run (excluded from stats)
       │    │    ├─ For run 1..10:
       │    │    │    ├─ Create adapter
       │    │    │    ├─ Attach metric collectors (TTFF, drops, CPU, memory, seek, rebuffer, binary)
       │    │    │    ├─ Open → wait for prepared → Play
       │    │    │    ├─ Steady-state playback (30s) with per-frame metric updates
       │    │    │    ├─ Seek tests (VOD only: 25%, 50%, 75% x 5 seeks each)
       │    │    │    ├─ Stop → collect results → destroy adapter
       │    │    │    └─ GC.Collect → 2s cooldown
       │    │    └─ Aggregate 10 runs → median, mean, stddev, P5, P95
       │    └─ Save per-scenario results
       └─ Export JSON + CSV
```

Each `IMetricCollector` receives three callbacks:
- `OnRunStart(adapter)` — reset and start measuring
- `OnUpdate(adapter, deltaTime)` — called every frame during playback
- `OnRunEnd(adapter)` — finalize measurements

Results are collected via `GetResults()` which returns a flat `Dictionary<string, double>` of named metric values. The runner aggregates these across runs using `StatisticsHelper`.
