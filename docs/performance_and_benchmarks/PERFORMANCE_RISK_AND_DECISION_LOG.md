# PERFORMANCE RISK & DECISION LOG
## Document ID: KW-PRF-005 | Phase 2: System Architecture & Engineering Standards

---

### 1. Architectural Component Bottleneck Analysis

To prevent unexpected performance regressions during implementation, every core architectural component has been audited for potential bottlenecks, measurement uncertainties, and failure modes:

```
+---------------------------------------------------------------------------------------------------------+
|                                    COMPONENT BOTTLENECK ANALYSIS                                        |
+--------------------------+---------------------------------+--------------------------------------------+
| SUBSYSTEM / COMPONENT    | POTENTIAL BOTTLENECK            | ARCHITECTURAL MITIGATION                   |
+--------------------------+---------------------------------+--------------------------------------------+
| Windows Desktop Hook     | DWM compositing stall on WorkerW| Zero-copy Direct3D swapchain presentation  |
| Process Supervision      | CPU burn from polling processes | Event-driven SetWinEventHook + IPC events  |
| Video Playback Engine    | Software decode fallback on iGPU| D3D11VA hardware decode assert + AV1/VP9   |
| Web Wallpaper Engine     | WebView2 VRAM / RAM bloat       | Virtual host mapping + Low Integrity sandbx|
| Multi-Monitor Management | Mixed-DPI & mixed-refresh desync| Independent player process per monitor     |
| WASAPI Audio Visualizer  | High CPU load from FFT analysis | 16-32 frequency bucket downsampling @ 30Hz |
| Thumbnail Generation     | Disk thrashing during imports   | Bounded threadpool + pure Rust WebP encode |
| State Persistence        | Disk write stalls on config save| Asynchronous Win32 MoveFileExW replace     |
+--------------------------+---------------------------------+--------------------------------------------+
```

---

### 2. Inherited Risk Register Mappings (from Step 6)

The performance specifications and test suites directly mitigate risks identified in **Step 6 (Formal Risk Register, KW-R-001..020)**:

| Risk ID | Risk Description | Inherent Severity | Test Mitigation ID | Technical Control Implemented |
| :--- | :--- | :--- | :--- | :--- |
| **`KW-R-001`** | High background CPU usage from unoptimized polling loops. | **Critical (16)** | `PERF-001`, `REGR-001` | WinEvent event-driven architecture; sleep loops banned (`DEC-PRF-01`). |
| **`KW-R-002`** | Unbounded native memory leaks during continuous playback. | **Critical (16)** | `PERF-007` (24h Soak)| Automated 24-hour leak detection; private byte growth capped $\le 2.0\text{ MB}$. |
| **`KW-R-004`** | WorkerW desktop attachment failure on Windows 11 updates. | **High (12)** | `LIFE-003`, `STAB-001`| Dual-mode attachment fallback (WorkerW + Progman hook); clean desktop recovery. |
| **`KW-R-006`** | Stutter and dropped frames in competitive 3D games. | **High (12)** | `PERF-004`, `PERF-005`| Sub-100ms pause reaction; drops GPU/CPU decode overhead to $0.00\%$. |
| **`KW-R-013`** | Crash or freeze during dynamic display disconnect. | **High (12)** | `INT-002`, `STAB-001` | Independent player processes; crashing display does not bring down other monitors. |
| **`KW-R-014`** | Excessive battery consumption on mobile laptops. | **High (12)** | `PERF-008` (Battery)  | Automatic pause or 30fps throttle on DC battery power (`DEC-PRF-04`). |

---

### 3. Open Performance Decision Register (`DEC-PRF-01` .. `08`)

```
+--------------------------------------------------------------------------------------------------------------------------+
|                                           PERFORMANCE DECISION REGISTER (STEP 12)                                        |
+-------------+-------------------------------------------------------+----------+-----------------------------------------+
| DECISION ID | TITLE / SUMMARY                                       | STATUS   | ARCHITECTURAL RATIONALE                 |
+-------------+-------------------------------------------------------+----------+-----------------------------------------+
| DEC-PRF-01  | Event-Driven Win32 Hooks over High-Frequency Polling  | APPROVED | Drops idle CPU to <= 0.10%, zero spin   |
| DEC-PRF-02  | Ephemeral UI Architecture (Zero Resident RAM on Close)| APPROVED | Frees 55 MB RAM immediately on close    |
| DEC-PRF-03  | Direct3D11VA Hardware Video Decoding Swapchains       | APPROVED | Offloads video to GPU ASIC, < 2% CPU    |
| DEC-PRF-04  | Dynamic Player Process Suspension on Fullscreen       | APPROVED | Zero game contention, sub-100ms pause   |
| DEC-PRF-05  | Asynchronous Multi-Process Startup Staggering         | APPROVED | Prevents disk / CPU boot contention     |
| DEC-PRF-06  | Downsampled In-Memory FFT Audio Pipeline              | APPROVED | < 1.5% CPU, zero raw PCM disk writes    |
| DEC-PRF-07  | Bounded WebP / Shader LRU Cache (256 MB Quota)        | APPROVED | Prevents unbounded disk consumption     |
| DEC-PRF-08  | Windows Job Object KILL_ON_JOB_CLOSE Teardown         | APPROVED | Guarantees zero zombie processes on exit|
+-------------+-------------------------------------------------------+----------+-----------------------------------------+
```

#### Detailed Decision Records:

- **DEC-PRF-01: Event-Driven Win32 Hooks over High-Frequency Polling**
  - *Context*: Detecting foreground window changes, fullscreen games, display topology shifts, and power events.
  - *Decision*: Prohibit timer-based polling loops. Utilize native Win32 event hooks: `SetWinEventHook` for window focus, `RegisterPowerSettingNotification` for battery/AC transitions, and `WM_DISPLAYCHANGE` for monitor topology.
  - *Consequences*: Core Controller wake-up rate drops to $\le 4\text{ Hz}$ when idle, achieving $\le 0.10\%$ background CPU utilization.

- **DEC-PRF-02: Ephemeral UI Architecture**
  - *Context*: Managing memory footprint of the Tauri v2 + React gallery interface.
  - *Decision*: Terminate `kinetiwall-ui.exe` upon window close. State is persisted in `config.json` by the Core daemon.
  - *Consequences*: Completely eliminates the 50–90 MB resident RAM penalty suffered by competitors whose UI remains idling in the background.

- **DEC-PRF-03: Direct3D11VA Hardware Video Decoding Swapchains**
  - *Context*: Rendering 1080p and 4K video frames to the desktop.
  - *Decision*: Configure `libmpv` to decode video via Direct3D11 Video Acceleration (`--hwdec=d3d11va`) and present directly into a native D3D11 swapchain.
  - *Consequences*: Eliminates CPU memory blits and color space conversions. CPU utilization during 1080p60 playback drops from 12–18% (software) to $< 2.0\%$ (hardware).

- **DEC-PRF-04: Dynamic Player Process Suspension on Fullscreen**
  - *Context*: Preventing live wallpaper rendering from competing with 3D games or video editing applications.
  - *Decision*: When a fullscreen or maximized window occludes the wallpaper, the Core Controller commands the player to pause Direct3D swapchain presentation immediately ($\le 100\text{ ms}$).
  - *Consequences*: Video decode engine utilization and presentation GPU utilization drop to exactly $0.00\%$. Zero FPS degradation on user games.

- **DEC-PRF-05: Asynchronous Multi-Process Startup Staggering**
  - *Context*: Avoiding disk and CPU contention during Windows logon when multiple monitors are connected.
  - *Decision*: Core initializes system tray and IPC first ($\le 800\text{ ms}$), then spawns player subprocesses sequentially with a 50ms stagger.
  - *Consequences*: Smooth system startup without desktop stutter or high disk queue lengths.

- **DEC-PRF-06: Downsampled In-Memory FFT Audio Pipeline**
  - *Context*: Providing real-time audio visualization without audio buffer latency or high CPU load.
  - *Decision*: Capture WASAPI loopback audio, perform FFT in-memory via `rustfft`, downsample into 16 or 32 normalized floating-point energy buckets, and deliver to web visualizers at 30 or 60 Hz.
  - *Consequences*: CPU utilization remains $< 1.5\%$. Mathematically impossible to reconstruct speech or private audio (satisfying `KW-F-024` privacy requirements).

- **DEC-PRF-07: Bounded WebP / Shader LRU Cache (256 MB Quota)**
  - *Context*: Caching generated thumbnails and compiled DirectX shader blobs.
  - *Decision*: Implement an automatic Least Recently Used (LRU) eviction algorithm triggered whenever total cache exceeds 256 MB, pruning oldest files down to 204 MB (80% watermark).
  - *Consequences*: Prevents uncontrolled cache expansion on user drives over months of usage.

- **DEC-PRF-08: Windows Job Object KILL_ON_JOB_CLOSE Teardown**
  - *Context*: Preventing orphaned player subprocesses from consuming resources if the Core Controller crashes or exits.
  - *Decision*: Assign all spawned player subprocesses to a Windows Job Object configured with `JOB_OBJECT_LIMIT_KILL_ON_JOB_CLOSE`.
  - *Consequences*: The Windows NT kernel automatically and cleanly terminates all child processes when the Core controller process handle closes, ensuring zero zombie processes.

---

### 4. Remaining Candidate Target Validation Gates

The following performance targets remain provisional until verified against executable builds in Phase 3 and Phase 5:

1. **`KW-MET-018` (4K30 Video CPU Ceiling $\le 4.5\%$)**:
   - *Validation Gate*: Requires testing across older integrated GPUs (Intel UHD 630 / 730) where memory bus bandwidth limits 4K throughput.
2. **`KW-MET-019` (WebGL Complex Shader CPU Ceiling $\le 4.0\%$)**:
   - *Validation Gate*: Dependent on Microsoft Edge WebView2 Chromium release optimizations and GPU driver WebGL implementations.
3. **`KW-MET-020` (Audio FFT Processing $\le 1.5\%$ on Single Core)**:
   - *Validation Gate*: Requires testing on entry-level dual-core mobile processors.

