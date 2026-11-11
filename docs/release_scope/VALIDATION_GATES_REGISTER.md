# KinetiWall — Validation Gates & Proof-of-Concept Register
**Document ID:** `KW-SCP-006`  
**Deliverable:** Deliverable F — Proof-of-Concept and Validation Gate Register  
**Phase:** 1 — Project Research, Requirements & Feasibility  
**Step:** 5 of 90 — Release Scope, Non-Goals & Feature Prioritization  
**Status:** COMPLETE (Auditable Gate Specifications)  
**Authority:** Systems Architecture & Quality Assurance  
**Date:** 2026-11-10  

---

## 1. Purpose & Risk-Reduction Philosophy

Before committing engineering resources to extensive production implementation in Phase 3, high-risk technical assumptions must pass formal **Validation Gates (VG)**. 

A Validation Gate is an empirical prototype spike executed against concrete pass/fail criteria in a controlled environment. If a gate fails, the project triggers an immediate architectural review rather than compounding technical debt.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    VALIDATION GATES EXECUTION PIPELINE                      │
├─────────┬───────────────────────────────────┬──────────────┬────────────────┤
│ Gate ID │ Technical Domain Spike            │ Risk Profile │ Phase Target   │
├─────────┼───────────────────────────────────┼──────────────┼────────────────┤
│ VG-001  │ Desktop Attachment & Restoration  │ CRITICAL     │ Phase 2 (P0)   │
│ VG-002  │ Job Object Process Cleanup        │ HIGH         │ Phase 2 (P0)   │
│ VG-003  │ libmpv D3D11VA Gapless Looping    │ HIGH         │ Phase 2 (P0)   │
│ VG-004  │ Event Fullscreen Detection Pause  │ MEDIUM       │ Phase 2 (P0)   │
│ VG-005  │ Multi-Monitor Topology Recovery   │ HIGH         │ Phase 2 (P1)   │
│ VG-006  │ WebView2 Virtual Host Sandboxing  │ CRITICAL     │ Phase 2 (P0)   │
│ VG-007  │ WASAPI Volatile FFT Audio Bridge  │ MEDIUM       │ Phase 2 (P1)   │
│ VG-008  │ Offline Installer Prerequisites   │ MEDIUM       │ Phase 2 (P0)   │
│ VG-009  │ Native Executable Reparenting     │ VERY HIGH    │ Phase 2 (P3)   │
│ VG-010  │ Media Library LGPL Compliance     │ LEGAL/TECH   │ Phase 2 (P0)   │
└─────────┴───────────────────────────────────┴──────────────┴────────────────┘
```

---

## 2. Exhaustive Validation Gate Specifications

### Gate 1: `VG-001` — Windows Desktop Shell Attachment & Restoration
- **Core Question:** Can KinetiWall reliably attach a rendering surface behind desktop icons across Windows 11 24H2 and Windows 10 22H2 without visual artifacting or icon event blocking, and cleanly restore default wallpaper on exit?
- **Candidate Technique:** Message `0x052C` to `Progman` to spawn `WorkerW`; Dual-Mode fallback anchoring to `Progman` child beneath `SHELLDLL_DefView` when desktop icons are hidden (`DEC-PLT-001`).
- **Smallest Valid Experiment:** Standalone 150-line Rust CLI tool (`spike-desktop-host`) that queries desktop window hierarchy, sends `0x052C`, locates target `HWND`, creates a Direct2D colored child window, tests desktop icon interaction, toggles desktop icon visibility, and detaches cleanly on Ctrl+C.
- **Required Environment:** Windows 11 24H2 (x64) and Windows 10 22H2 (x64) clean virtual machines.
- **Evidence to Collect:** Spy++ window hierarchy tree dumps, screen recordings of desktop icon dragging over rendering window, process handle count.
- **Success Criteria:** 
  1. Direct2D surface renders behind desktop icons.
  2. All desktop icons remain 100% clickable, selectable, and draggable.
  3. When "Show desktop icons" is unchecked on Win 11 24H2, rendering surface remains visible via fallback mode.
  4. On process exit, original desktop wallpaper restores in <200ms with zero black screen.
- **Failure Criteria:** Desktop icons become unclickable; rendering surface covers icons; unchecking icons causes black screen; Explorer crashes.
- **Scope Consequence if Failed:** Mandatory architectural review. Fallback to transparent borderless layered window (`WS_EX_LAYERED | WS_EX_TRANSPARENT`) or user notification of limitation.
- **Approval Required for Changes:** User approval required.

---

### Gate 2: `VG-002` — Player Process Supervision & Orphan Cleanup
- **Core Question:** Does binding out-of-process wallpaper players to a Windows Job Object guarantee 100% orphan-process termination when the core controller exits or crashes abruptly?
- **Candidate Technique:** Win32 Job Objects configured with `JOB_OBJECT_LIMIT_KILL_ON_JOB_CLOSE` and Named Pipe IPC heartbeat monitoring.
- **Smallest Valid Experiment:** Standalone Rust binary that creates a Job Object, spawns 3 dummy child processes that write timestamps to stdout, binds them to the job, and performs hard termination (`std::process::abort()`).
- **Required Environment:** Windows 11 (x64) and Windows 10 (x64).
- **Evidence to Collect:** PowerShell `Get-Process` verification showing zero lingering child PIDs immediately after controller crash.
- **Success Criteria:** 100% of child processes terminate within 50ms of controller termination across 50 consecutive test iterations.
- **Failure Criteria:** Any child process continues executing as an orphaned background task.
- **Scope Consequence if Failed:** P0 blocker. Must implement secondary watchdog daemon or fallback IPC graceful teardown protocol.
- **Approval Required for Changes:** Internal architectural resolution.

---

### Gate 3: `VG-003` — libmpv Hardware Video Decoding & Gapless Looping
- **Core Question:** Does dynamically linked `libmpv-2.dll` provide stable Direct3D 11 hardware decoding (`vo=gpu-next`, `hwdec=d3d11va`) with sub-16.6ms gapless looping and instant pause on Windows?
- **Candidate Technique:** C-API dynamic binding to `libmpv-2.dll` from Rust (`libmpv-sys`), rendering into a win32 child `HWND` with `loop-file=inf`.
- **Smallest Valid Experiment:** Minimal Rust CLI player loading 4K H.264, 4K HEVC, and 4K AV1 video samples in an infinite loop, measuring CPU/GPU usage and frame-time delta at loop boundary via ETW tracing.
- **Required Environment:** Windows 11 x64 PC with dedicated GPU (NVIDIA/AMD) and integrated GPU (Intel Iris/UHD).
- **Evidence to Collect:** GPU-Z hardware decode engine activity graph, Windows Performance Recorder CPU profile, high-speed camera frame analysis.
- **Success Criteria:**
  1. GPU video decode engine active (D3D11VA verified).
  2. CPU usage <1.5% during 4K video playback.
  3. Gap between final frame and first frame at loop boundary ≤16.6ms with zero black frame flash.
  4. Halting playback (`pause=yes`) immediately drops player GPU load to 0.0%.
- **Failure Criteria:** Software fallback decoding (>15% CPU load); visual black frame flash at loop point; crash on video load.
- **Scope Consequence if Failed:** If libmpv fails on specific hardware, evaluate Windows Media Foundation (WMF) D3D11 rendering backend as secondary fallback.
- **Approval Required for Changes:** Architecture review.

---

### Gate 4: `VG-004` — Event-Driven Fullscreen Detection & Gaming Impact
- **Core Question:** Can Win32 event hooks (`SetWinEventHook`) detect fullscreen games and maximized applications within 50ms without polling, and does pausing wallpapers achieve 0.00% gaming interference?
- **Candidate Technique:** Hooking `EVENT_SYSTEM_FOREGROUND` and `EVENT_OBJECT_LOCATIONCHANGE`, comparing foreground window bounds with monitor rects.
- **Smallest Valid Experiment:** Event-listener service that logs timestamped pause/resume events when fullscreen DirectX games (e.g. Counter-Strike 2, Cyberpunk 2077) or maximized windows gain/lose focus.
- **Required Environment:** Windows 11 x64 gaming PC with dual monitors.
- **Evidence to Collect:** CapFrameX frame-time logs comparing 99th percentile FPS during game execution with KinetiWall paused vs KinetiWall completely shut down.
- **Success Criteria:**
  1. Fullscreen game detection latency <50ms from window focus change.
  2. Paused wallpaper player drops to 0.00% CPU and 0.00% GPU utilization.
  3. Game 99th percentile frame-time variance is <0.5% (zero measurable FPS penalty).
  4. Zero polling threads active in background.
- **Failure Criteria:** Failure to detect exclusive fullscreen or borderless windowed games; game FPS drop >1.0%; engine spins CPU threads.
- **Scope Consequence if Failed:** P0 blocker. Refine window style inspection logic (`GetWindowLongPtrW(GWL_STYLE)` checking `WS_POPUP`).
- **Approval Required for Changes:** Internal resolution.

---

### Gate 5: `VG-005` — Multi-Monitor Dynamic Topology & DPI Scaling
- **Core Question:** How reliably does the engine handle monitor hotplugging, resolution switching, mixed-DPI scaling, and display sleep/wake without crashing or misaligning windows?
- **Candidate Technique:** Windows message loop capturing `WM_DISPLAYCHANGE`, `WM_DPICHANGED`, and `WM_WTSSESSION_CHANGE` with Per-Monitor V2 DPI awareness.
- **Smallest Valid Experiment:** Test harness hosting independent Direct2D color surfaces across two physical displays with differing scale factors (100% and 150%), triggering simulated resolution and topology changes.
- **Required Environment:** Multi-monitor Windows 11 setup with mixed DPI scaling.
- **Evidence to Collect:** Screenshot logs after display changes, window coordinate inspection.
- **Success Criteria:**
  1. Disconnecting secondary display cleanly migrates or destroys child player without crashing core engine.
  2. Reconnecting display re-instantiates assigned wallpaper within 1.0s.
  3. Changing DPI dynamically re-scales rendering viewport with zero coordinate drift or blurring.
- **Failure Criteria:** Window offset off-screen; duplicate player processes spawned; engine crash on `WM_DISPLAYCHANGE`.
- **Scope Consequence if Failed:** Refine Win32 monitor rect cache and re-enumeration pipeline.
- **Approval Required for Changes:** Internal resolution.

---

### Gate 6: `VG-006` — WebView2 Origin Sandboxing & Local Resource Restrictions
- **Core Question:** Can Microsoft Edge WebView2 host interactive HTML5/WebGL wallpapers via Virtual Host Folder Mapping while strictly blocking external network requests, camera/mic permissions, and unauthorized local disk access?
- **Candidate Technique:** `SetVirtualHostNameToFolderMapping` mapped to `https://kinetiwall.local/` with `COREWEBVIEW2_HOST_RESOURCE_ACCESS_KIND_DENY_CORS` and event listeners blocking `PermissionRequested` and `NewWindowRequested`.
- **Smallest Valid Experiment:** Standalone WebView2 host executing a malicious test suite HTML page attempting: (a) external image/script fetch, (b) local file traversal (`../../Windows/win.ini`), (c) microphone/camera access, (d) pop-up spawn (`window.open`).
- **Required Environment:** Windows 10/11 x64 with Microsoft Edge WebView2 Evergreen Runtime.
- **Evidence to Collect:** WebView2 security event logs, browser console output, network packet logs.
- **Success Criteria:**
  1. All external HTTP/HTTPS network requests fail with CORS/network error.
  2. Local file directory traversal blocked outside mapped wallpaper folder.
  3. Hardware camera/mic/location requests denied unconditionally.
  4. Popups (`window.open`) blocked.
  5. High-complexity WebGL 2.0 shaders execute smoothly at 60 FPS.
- **Failure Criteria:** Any external network packet escapes; local file outside folder is readable; popup window opens.
- **Scope Consequence if Failed:** CRITICAL security failure. Web player cannot be released without passing this gate.
- **Approval Required for Changes:** User approval required.

---

### Gate 7: `VG-007` — WASAPI Volatile Loopback Audio Capture & FFT Processing
- **Core Question:** Can Windows WASAPI loopback capture system audio with zero playback interference, process 1024-point FFT in pure Rust, and stream frequency data to players in <20ms while consuming <0.3% CPU?
- **Candidate Technique:** `IAudioClient` loopback initialization on default multimedia device, pure-Rust SIMD-accelerated FFT (`realfft`), pushing frequency arrays over shared memory.
- **Smallest Valid Experiment:** CLI tool capturing system audio during Spotify playback, calculating 64-band frequency magnitudes at 60 FPS, and verifying volatile memory constraints.
- **Required Environment:** Windows 11/10 with standard Realtek / USB audio devices.
- **Evidence to Collect:** Audio latency measurements, CPU profiler traces, disk I/O activity logs confirming 0 bytes written.
- **Success Criteria:**
  1. Capture stream synchronized with audible sound (<20ms latency).
  2. CPU usage strictly <0.3% during sustained 60 FPS FFT calculation.
  3. Zero audio stuttering, clicks, or popping introduced into system playback.
  4. Audio device change (`WM_DEVICECHANGE`) triggers clean capture re-attachment within 500ms.
- **Failure Criteria:** Audio artifacting in system speakers; high CPU consumption (>1.0%); crash on headphone unplug.
- **Scope Consequence if Failed:** Defer audio visualizer to V1.1; core video/web wallpaper engine unaffected.
- **Approval Required for Changes:** Internal architectural resolution.

---

### Gate 8: `VG-008` — Standalone Offline Installer & Prerequisite Verification
- **Core Question:** Can a self-contained NSIS/WiX installer successfully install, configure, and execute KinetiWall on an air-gapped clean Windows machine with zero internet connectivity?
- **Candidate Technique:** Standalone executable bundle packing engine binaries, local `libmpv-2.dll`, WebView2 Evergreen bootstrapper/offline runtime checks, and VC++ redistributable detection.
- **Smallest Valid Experiment:** Build mock NSIS installer package, copy via USB to a freshly installed, air-gapped Windows 10 22H2 virtual machine with network adapter disabled, execute installation, and launch engine.
- **Required Environment:** Air-gapped Windows 10 22H2 clean install (no pre-installed third-party runtimes).
- **Evidence to Collect:** Installer exit code, application launch verification, dependency check logs.
- **Success Criteria:**
  1. Installer completes without requesting internet connection.
  2. Engine launches and displays default wallpaper immediately.
  3. Uninstaller cleanly removes all application files and shortcuts.
- **Failure Criteria:** Installer fails due to missing online dependencies; application crashes on launch due to missing DLLs.
- **Scope Consequence if Failed:** Bundle static VC++ runtime or provide clear offline redistributables package.
- **Approval Required for Changes:** Internal resolution.

---

### Gate 9: `VG-009` — Native Executable Reparenting & Safety Sandbox
- **Core Question:** Can a compiled standalone Win32 game executable (Godot/Unity .exe) be reparented into `WorkerW` without stealing focus from active applications or causing mouse cursor traps?
- **Candidate Technique:** Spawning external `.exe` as suspended process, applying `WS_CHILD | WS_CLIPSIBLINGS` window styles via `SetWindowLongPtrW`, reparenting via `SetParent`, and trapping mouse input via `SetWindowsHookEx`.
- **Smallest Valid Experiment:** Prototype harness attempting to embed a compiled Godot 4 demo game behind desktop icons.
- **Required Environment:** Windows 11 x64.
- **Evidence to Collect:** Focus loss logs, input contention traces, crash dump analysis.
- **Success Criteria:**
  1. Executable window renders behind desktop icons.
  2. Foreground applications maintain focus without game window stealing keyboard/mouse focus.
  3. Killing controller cleanly terminates the reparented executable.
- **Failure Criteria:** Game window steals focus every frame; mouse becomes trapped; game crashes on resize; anti-cheat flags process reparenting.
- **Scope Consequence if Failed:** CONFIRMS EXCLUSION FROM V1.0. Native standalone `.exe` reparenting remains deferred to V2.0+; V1.0 delivers 3D via WebGL in WebView2 (`KW-F-019`).
- **Approval Required for Changes:** User approval required before altering V1.0 scope.

---

### Gate 10: `VG-010` — Media Library Dynamic Linkage & LGPL Redistribution
- **Core Question:** Does KinetiWall's dynamic linkage to `libmpv-2.dll` completely satisfy Section 6 of the GNU Lesser General Public License (LGPL v2.1+), preventing any copyleft infection of KinetiWall's proprietary or MIT/Apache-licensed Rust codebase?
- **Candidate Technique:** Dynamic linking via runtime DLL loading (`LoadLibraryW` / `GetProcAddress` or standard DLL import library linking), shipping unmodified upstream `libmpv-2.dll` binaries alongside license texts and source code download links.
- **Smallest Valid Experiment:** Independent verification test: replace the shipping `libmpv-2.dll` with an independently compiled custom debug build of libmpv; verify KinetiWall launches, detects, and executes using the replacement library with zero recompilation of KinetiWall binaries.
- **Required Environment:** Windows 10/11 developer workstation.
- **Evidence to Collect:** Dependency Walker / PE header audit, successful video playback with custom-compiled DLL, legal compliance documentation bundle.
- **Success Criteria:**
  1. KinetiWall binary PE import tables confirm dynamic linking to `libmpv-2.dll`.
  2. Replacing `libmpv-2.dll` with an arbitrary ABI-compatible build executes successfully without re-linking KinetiWall.
  3. Full license documentation, copyright notices, and source code build instructions included in installer.
- **Failure Criteria:** Static linking detected; proprietary code infected by copyleft GPL; DLL replacement fails.
- **Scope Consequence if Failed:** CRITICAL LEGAL BLOCKER. Must re-architect build configuration to ensure 100% dynamic linkage.
- **Approval Required for Changes:** User approval required.

