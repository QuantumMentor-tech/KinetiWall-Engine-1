# PERFORMANCE BENCHMARK & TEST MATRIX
## Document ID: KW-PRF-002 | Phase 2: System Architecture & Engineering Standards

---

### 1. Test Architecture & Coverage Framework

The KinetiWall Performance & Verification Test Matrix organizes all validation scenarios into a six-tier pyramid, ensuring that unit-level logic, inter-process communication contracts, end-to-end user flows, sustained stress loads, regressions, and crash fault tolerances are empirically validated.

```
+---------------------------------------------------------------------------------------------------------+
|                                    KINETIWALL TEST SUITE PYRAMID                                        |
+--------+----------------------------+-------------+-----------------------------------------------------+
| TIER   | TEST CATEGORY              | ID PREFIX   | FOCUS & EXECUTION CADENCE                           |
+--------+----------------------------+-------------+-----------------------------------------------------+
| Tier 1 | Unit Tests                 | UNIT-xxx    | Deterministic logic, policy engines (Per Commit CI) |
| Tier 2 | Integration Tests          | INT-xxx     | IPC timing, Named Pipe contracts (Per PR CI)        |
| Tier 3 | System & Lifecycle Tests   | LIFE-xxx    | Full process tree boot, UI close, teardown (Nightly)|
| Tier 4 | Performance & Stress Tests | PERF-xxx    | 24h soak, 4K60 media, multi-monitor (Nightly / Gate)|
| Tier 5 | Performance Regression     | REGR-xxx    | Automated threshold delta checking (Per Release)    |
| Tier 6 | Reliability & Security     | STAB-xxx    | Crash injection, WorkerW recovery, IPC floods (Gate)|
+--------+----------------------------+-------------+-----------------------------------------------------+
```

> [!NOTE]
> All test records in this matrix are currently classified as **`NOT RUN (DESIGN PHASE)`**. Physical test execution will commence upon completion of the implementation phases (Phases 3–7).

---

### 2. Tier 1: Performance Unit Tests (`UNIT-001` .. `004`)

#### `UNIT-001`: Pause Policy Decision Latency
- **Requirement Ref**: `KW-NF-004`, `ADR-0005` | **Status**: **NOT RUN (DESIGN PHASE)**
- **Preconditions**: Isolated mock test harness; zero external Win32 API calls.
- **Test Stimulus**: Invoke `EvaluationEngine::evaluate_pause_policy()` with simulated fullscreen foreground window event.
- **Duration / Repetitions**: 10,000 iterations in microbenchmark (`criterion`).
- **Metrics Collected**: Mean execution latency, P99 execution latency.
- **Pass Condition**: Mean latency $\le 5.0\ \mu\text{s}$, P99 latency $\le 20.0\ \mu\text{s}$.
- **Fail Condition**: Mean latency $> 50.0\ \mu\text{s}$ or memory allocations detected during evaluation.
- **Automation**: Fully automated in Cargo CI test suite.

#### `UNIT-002`: IPC Framing & Length Prefix Validation Bounds
- **Requirement Ref**: `KW-SEC-005`, `CTR-01` | **Status**: **NOT RUN (DESIGN PHASE)**
- **Test Stimulus**: Feed stream decoder with valid envelopes, oversized length prefixes ($> 4\text{ MB}$), and malformed JSON bytes.
- **Pass Condition**: Valid envelopes parse in $< 50\ \mu\text{s}$; oversized envelopes rejected immediately with zero heap allocation.

#### `UNIT-003`: Dynamic Property Value Clamping & Interpolation
- **Requirement Ref**: `KW-F-018`, `KW-DAT-004` | **Status**: **NOT RUN (DESIGN PHASE)**
- **Test Stimulus**: Pass slider float values above `max` and below `min` into `PropertyValidator`.
- **Pass Condition**: Values correctly clamped to valid range; zero panics or unhandled errors.

#### `UNIT-004`: Audio Spectrum FFT Bucket Downsampling Performance
- **Requirement Ref**: `KW-F-024`, `DEC-SEC-06` | **Status**: **NOT RUN (DESIGN PHASE)**
- **Test Stimulus**: Execute 1024-point FFT + 16-bucket downsampling across 1,000 simulated PCM audio frames.
- **Pass Condition**: Single frame processing time $\le 150\ \mu\text{s}$ on single core; zero memory allocations inside hot loop.

---

### 3. Tier 2: IPC & Subsystem Integration Tests (`INT-001` .. `004`)

#### `INT-001`: Core-to-Player Playback State Transition Latency
- **Requirement Ref**: `KW-NF-004`, `CTR-02` | **Status**: **NOT RUN (DESIGN PHASE)**
- **Preconditions**: `kinetiwall-core.exe` and mock player running connected via Named Pipe.
- **Test Stimulus**: Core transmits `CMD_PLAYBACK_CONTROL(pause)` over pipe.
- **Metrics Collected**: Time from `tokio::io::AsyncWrite` on Core to `EVT_PLAYBACK_STATE_CHANGED(paused)` read.
- **Pass Condition**: Round-trip IPC latency $\le 15.0\text{ ms}$ (P95).
- **Fail Condition**: Round-trip latency $> 50.0\text{ ms}$ or pipe buffer saturation.

#### `INT-002`: Display Topology Hook Disconnection Event Handling
- **Requirement Ref**: `KW-F-020`, `ADR-0002` | **Status**: **NOT RUN (DESIGN PHASE)**
- **Test Stimulus**: Simulate Win32 `WM_DISPLAYCHANGE` notification with secondary monitor detached.
- **Pass Condition**: Core suspends associated player within $100\text{ ms}$; leaves primary display uninterrupted.

#### `INT-003`: Ephemeral UI IPC Handshake on Cold Launch
- **Requirement Ref**: `KW-F-038`, `KW-NG-001` | **Status**: **NOT RUN (DESIGN PHASE)**
- **Test Stimulus**: Launch `kinetiwall-ui.exe`; perform initial Named Pipe connection and state synchronization query.
- **Pass Condition**: Complete state synchronization achieved in $\le 50.0\text{ ms}$.

#### `INT-004`: Atomic Config Persistence & Read Latency
- **Requirement Ref**: `KW-DAT-007`, `DEC-DATA-03` | **Status**: **NOT RUN (DESIGN PHASE)**
- **Test Stimulus**: Save updated monitor layout to `config.json` via Win32 `MoveFileExW` atomic replace.
- **Pass Condition**: Total write + flush + atomic rename latency $\le 20.0\text{ ms}$; zero file corruption.

---

### 4. Tier 3: System Lifecycle Tests (`LIFE-001` .. `006`)

#### `LIFE-001`: First-Run Engine Bootstrap & Baseline Initialization
- **Requirement Ref**: `KW-NF-006` | **Status**: **NOT RUN (DESIGN PHASE)**
- **Preconditions**: Clean user profile with no existing `%LOCALAPPDATA%\KinetiWall` directory.
- **Action**: Launch `kinetiwall-core.exe`.
- **Expected Outcome**: Directory tree created, default `config.json` written, system tray icon visible, IPC listener ready.
- **Pass Condition**: Total elapsed boot time $\le 800\text{ ms}$; peak memory $\le 25.0\text{ MB}$.

#### `LIFE-002`: Standard Cold Boot on Windows Logon
- **Requirement Ref**: `KW-NF-006`, `KW-F-030` | **Status**: **NOT RUN (DESIGN PHASE)**
- **Action**: Simulate Windows logon startup shortcut trigger with assigned video wallpaper.
- **Pass Condition**: Tray icon responsive $\le 800\text{ ms}$; first video frame presented $\le 1,200\text{ ms}$.

#### `LIFE-003`: Wallpaper Assignment & Desktop WorkerW Attachment
- **Requirement Ref**: `KW-F-006`, `ADR-0001` | **Status**: **NOT RUN (DESIGN PHASE)**
- **Action**: Assign `REF-VID-01` to primary display.
- **Pass Condition**: Player spawned, window reparented behind desktop icons (`WorkerW`), playback active in $\le 300\text{ ms}$. Desktop icons remain fully clickable and responsive.

#### `LIFE-004`: Ephemeral UI Open and Complete Teardown
- **Requirement Ref**: `KW-NG-001`, `KW-MET-003` | **Status**: **NOT RUN (DESIGN PHASE)**
- **Action**: Open Gallery UI from tray, browse library for 30s, click close button `[X]`.
- **Pass Condition**: `kinetiwall-ui.exe` process exits cleanly within $500\text{ ms}$; resident memory drops to exactly $0.0\text{ MB}$. Core Controller continues running uninterrupted at $\le 15.0\text{ MB}$.

#### `LIFE-005`: System Display Sleep & Resume Cycle
- **Requirement Ref**: `KW-NF-008`, `KW-F-028` | **Status**: **NOT RUN (DESIGN PHASE)**
- **Action**: Trigger Windows monitor power-off via `SendMessage(HWND_BROADCAST, WM_SYSCOMMAND, SC_MONITORPOWER, 2)`, wait 60s, resume.
- **Pass Condition**: Playback pauses immediately on display sleep; resumes seamlessly upon wake with zero DirectX device lost crashes.

#### `LIFE-006`: Clean Application Exit & Desktop Recovery
- **Requirement Ref**: `KW-F-008`, `ADR-0002` | **Status**: **NOT RUN (DESIGN PHASE)**
- **Action**: Select "Exit KinetiWall" from system tray menu.
- **Pass Condition**: All player subprocesses terminated within $250\text{ ms}$; `WorkerW` reparenting cleanly detached; original Windows static wallpaper instantly visible without redrawing artifacts or shell restart.

---

### 5. Tier 4: Performance & Stress Tests (`PERF-001` .. `012`)

```
+-------------------------------------------------------------------------------------------------------------------------+
|                                           PERFORMANCE & STRESS TEST MATRIX                                              |
+----------+------------------------------------+---------------------+------------------+--------------------------------+
| TEST ID  | TEST SCENARIO DESCRIPTION          | WORKLOAD / CONTENT  | DURATION / RUNS  | PASS THRESHOLD (METRIC REF)    |
+----------+------------------------------------+---------------------+------------------+--------------------------------+
| PERF-001 | Core Daemon 1-Hour Idle Baseline   | No active wallpaper | 3,600s (1 Hour)  | CPU <= 0.10%, RAM <= 15.0 MB   |
| PERF-002 | Cold Engine Boot Latency           | Standard install    | 10 Repetitions   | Mean <= 800ms, P95 <= 950ms    |
| PERF-003 | 1080p60 Video Playback Soak Test   | REF-VID-01 (AVC)    | 600s (10 Mins)   | CPU <= 2.0%, Drops <= 0.10%    |
| PERF-004 | Pause & Resume Latency Benchmark   | 100 Toggle Cycles   | 100 Cycles       | Pause <= 100ms, Resume <= 150ms|
| PERF-005 | Gaming FPS Contention Benchmark    | 3D Benchmark Active | 600s Game Run    | Game FPS Drop <= 0.50%         |
| PERF-006 | Maximized Window Occlusion CPU     | Word / Browser Max  | 300s Occluded    | Total KinetiWall CPU <= 0.10%  |
| PERF-007 | 24-Hour Continuous Memory Leak Soak| REF-VID-01 Loop     | 86,400s (24 Hrs) | RAM Delta (1h to 24h) <= 2.0 MB|
| PERF-008 | Mobile Battery Power Preservation  | DC Battery (Lap)    | 1,800s (30 Mins) | Discharge Delta <= 1.0W        |
| PERF-009 | Dual-Monitor Independent Playback  | TOP-02 (2x 1080p60) | 600s (10 Mins)   | Aggregate RAM <= 115 MB        |
| PERF-010 | 4K30 Ultra HD Video Playback       | REF-VID-02 (4K HEVC)| 600s (10 Mins)   | CPU <= 4.5%, HW Decode Active  |
| PERF-011 | WebGL Interactive Particle Stress  | REF-WEB-02 (Shaders)| 600s (10 Mins)   | Player RAM <= 85 MB, 60fps lock|
| PERF-012 | Rapid Wallpaper Switching Stress   | 50 Wallpapers Rapid | 50 Switches      | Zero handle leaks, Delta < 5MB |
+----------+------------------------------------+---------------------+------------------+--------------------------------+
```

---

### 6. Tier 5: Performance Regression Tests (`REGR-001` .. `004`)

#### `REGR-001`: Background Polling CPU Creep Detection
- **Target**: Detect accidental introduction of `std::thread::sleep` spinning loops in background tasks.
- **Pass Condition**: Core Controller CPU utilization must not increase by $> 0.05\%$ between Git commits.

#### `REGR-002`: Wallpaper Switching Memory Leak Regression
- **Target**: Ensure textures and Direct3D surface handles are freed when switching wallpapers.
- **Stimulus**: Cycle through 20 video wallpapers, resting 5 seconds on each.
- **Pass Condition**: Final private bytes after cycle 20 is within $\pm 3.0\text{ MB}$ of cycle 1.

#### `REGR-003`: Occlusion Pause Latency Degradation
- **Target**: Prevent UI thread or IPC contention from stalling pause reactions.
- **Pass Condition**: P95 pause latency remains strictly $\le 100\text{ ms}$.

#### `REGR-004`: Video Presentation Frame Drop Regression
- **Target**: Detect regressions in `libmpv` swapchain synchronization.
- **Pass Condition**: Dropped frame rate remains $\le 0.10\%$ across reference runs.

---

### 7. Tier 6: Reliability & Security Resilience Tests (`STAB-001` .. `005`)

#### `STAB-001`: Injected Player Process Hard Crash Isolation
- **Action**: Forcefully terminate `kinetiwall-player-video.exe` via `TerminateProcess` while rendering.
- **Pass Condition**: `kinetiwall-core.exe` survives 100% intact; logs failure; supervisor restarts player within $1,000\text{ ms}$; zero shell or desktop corruption.

#### `STAB-002`: Sudden Core Controller Termination (Job Object Cleanup)
- **Action**: Force-kill `kinetiwall-core.exe` via Task Manager.
- **Pass Condition**: Windows Job Object (`JOB_OBJECT_LIMIT_KILL_ON_JOB_CLOSE`) terminates all child player processes instantly ($< 50\text{ ms}$); zero orphaned processes remaining in background.

#### `STAB-003`: Named Pipe Flooding Denial-of-Service
- **Action**: Send 5,000 rapid messages per second across `\\.\pipe\kinetiwall-core-*`.
- **Pass Condition**: Core Controller memory remains bounded ($\le 25\text{ MB}$); queue throttles safely without crashing.

#### `STAB-004`: Malformed Media File Graceful Rejection
- **Action**: Attempt to load a truncated or corrupted 0-byte MP4 video.
- **Pass Condition**: Player reports `ERR_DECODER_FAILURE` cleanly; does not crash; desktop falls back to default wallpaper.

#### `STAB-005`: Web Wallpaper Memory Exhaustion (Heap Bomb)
- **Action**: Load web wallpaper containing `let arr = []; while(true) arr.push(new Array(1000000));`.
- **Pass Condition**: WebView2 renderer memory limit triggers clean tab crash; Core Controller isolates event and notifies user.

