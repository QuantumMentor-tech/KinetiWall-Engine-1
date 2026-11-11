# KinetiWall — Acceptance Criteria Catalog
**Document ID:** `KW-QAR-001`  
**Deliverable:** Deliverable A — Acceptance Criteria Catalog  
**Phase:** 1 — Project Research, Requirements & Feasibility  
**Step:** 6 of 90 — Acceptance Criteria, Success Metrics & Risk Register  
**Status:** COMPLETE (Auditable Verification Baseline)  
**Authority:** Technical Program Lead & QA Systems Architecture  
**Date:** 2026-11-10  

---

## 1. Specification Framework & Structure

Every acceptance criterion in this catalog follows an auditable 15-attribute specification model designed to eliminate ambiguity and enforce deterministic verification:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                 15-ATTRIBUTE ACCEPTANCE CRITERION STRUCTURE                 │
├─────────────────────┬───────────────────────────────────────────────────────┤
│ Attribute           │ Engineering Purpose & Standard                        │
├─────────────────────┼───────────────────────────────────────────────────────┤
│ 1. Unique ID        │ Canonical identifier (e.g., KW-AC-001).               │
│ 2. Requirement ID   │ Upstream traceable requirement (KW-F-xxx / KW-NF-xxx).│
│ 3. Title            │ Precise operational title.                            │
│ 4. Preconditions    │ Mandatory environment and engine initial state.       │
│ 5. Triggering Event │ Explicit stimulus or user action.                     │
│ 6. Expected Output  │ Deterministic, observable runtime behavior.           │
│ 7. Verification Method│ Automated test, ETW trace, or manual QA harness.    │
│ 8. Required Evidence│ Log file, screen recording, memory dump, or trace.    │
│ 9. Pass Condition   │ Unambiguous criteria required to mark test PASSED.    │
│ 10. Fail Condition  │ Conditions triggering immediate test failure.         │
│ 11. Severity        │ BLOCKER (P0), CRITICAL (P0), MAJOR (P1), MINOR (P2). │
│ 12. Configurations  │ Supported Windows OS builds & hardware architectures. │
│ 13. Dependencies    │ Upstream subsystem or prototype validation gates.     │
│ 14. Status          │ NOT RUN (PLANNED) / EXECUTED / BLOCKED.               │
│ 15. Authority       │ USER-REQUIRED / APPROVED-DECISION / RECOMMENDATION.   │
└─────────────────────┴───────────────────────────────────────────────────────┘
```

---

## 2. Domain 1: Desktop Shell Integration & Restoration

### `KW-AC-001`: Desktop Attachment behind Icons (Primary Mode)
- **Requirement ID:** `KW-F-001`
- **Title:** Primary `WorkerW` Surface Attachment Beneath Desktop Icons
- **Preconditions:** Windows desktop active; desktop icons set to visible ("Show desktop icons" checked); default Windows static wallpaper set.
- **Triggering Event:** Engine initialization; `kinetiwall-core.exe` broadcasts message `0x052C` to `Progman` and instantiates wallpaper player child window.
- **Expected Observable Behavior:** Wallpaper rendering surface renders directly behind `SHELLDLL_DefView`. Desktop icons remain 100% visible, clickable, selectable, and draggable. Taskbar remains in foreground.
- **Verification Method:** Win32 API window hierarchy audit via Spy++ / EnumChildWindows + automated UI mouse click test on desktop icons.
- **Required Evidence:** Spy++ window hierarchy dump showing `WorkerW` hosting player `HWND` as sibling beneath `SHELLDLL_DefView`; automated click coordinate log.
- **Pass Condition:** Player surface renders behind icons; clicking desktop icons launches associated files/apps; zero visual flickering.
- **Fail Condition:** Wallpaper renders in front of desktop icons; icons become unclickable; desktop icons flicker or disappear.
- **Severity:** `BLOCKER`
- **Configurations:** Windows 11 (24H2, 23H2, 22H2), Windows 10 (22H2–19041+); x86-64.
- **Dependencies:** `VG-001`, Win32 User32.
- **Status:** `NOT RUN (PLANNED)`
- **Authority:** `USER-REQUIRED`

### `KW-AC-002`: Windows 11 24H2 Hidden-Icons Fallback Attachment
- **Requirement ID:** `KW-F-002`, `DEC-PLT-001`
- **Title:** Dual-Mode Desktop Attachment Fallback when Icons are Hidden
- **Preconditions:** Windows 11 24H2; desktop context menu setting "Show desktop icons" is unchecked.
- **Triggering Event:** KinetiWall starts or user toggles "Show desktop icons" off.
- **Expected Observable Behavior:** Engine detects `WorkerW` detachment, immediately executes fallback anchoring directly beneath `Progman` child hierarchy behind `SHELLDLL_DefView`.
- **Verification Method:** Automated PowerShell script toggling icon visibility via shell COM object and checking rendering surface visibility.
- **Required Evidence:** DXGI present log confirming active frame presentation; desktop screenshot verifying visible wallpaper.
- **Pass Condition:** Wallpaper remains 100% visible and rendering when desktop icons are hidden; zero black screen.
- **Fail Condition:** Screen turns solid black; wallpaper window is hidden or destroyed; `0x052C` loop locks Explorer.
- **Severity:** `BLOCKER`
- **Configurations:** Windows 11 24H2 (Build 26100+); x86-64 & ARM64 (Prism).
- **Dependencies:** `KW-AC-001`, `DEC-PLT-001`.
- **Status:** `NOT RUN (PLANNED)`
- **Authority:** `APPROVED-DECISION`

### `KW-AC-003`: Explorer Shell Crash Recovery
- **Requirement ID:** `KW-F-003`
- **Title:** Automated Shell Re-Attachment Following `explorer.exe` Crash
- **Preconditions:** KinetiWall active and presenting live wallpaper on desktop.
- **Triggering Event:** `explorer.exe` process is terminated abruptly via `taskkill /F /IM explorer.exe` and restarts.
- **Expected Observable Behavior:** Core daemon detects `TaskbarCreated` registered window message, re-executes desktop attachment handshake within ≤2.5 seconds, and re-attaches active player surfaces without crashing.
- **Verification Method:** Automated test script killing and restarting `explorer.exe` while monitoring `kinetiwall-core.exe` PID and desktop surface state.
- **Required Evidence:** Engine log timestamp confirming `TaskbarCreated` receipt and re-attachment in <2,500ms; continuous engine uptime log.
- **Pass Condition:** Desktop wallpaper re-appears within 2.5s; core daemon does not crash; zero orphaned player windows.
- **Fail Condition:** Engine crashes; wallpaper remains detached or floating as a top-level window; re-attachment takes >2.5s.
- **Severity:** `CRITICAL`
- **Configurations:** Windows 11 (24H2, 23H2), Windows 10 (22H2); x86-64.
- **Dependencies:** `KW-AC-001`, Win32 `RegisterWindowMessageW`.
- **Status:** `NOT RUN (PLANNED)`
- **Authority:** `USER-REQUIRED`

### `KW-AC-004`: Clean Teardown & Static Desktop Restoration
- **Requirement ID:** `KW-F-004`
- **Title:** Desktop Surface Detachment and Wallpaper Restoration on Exit
- **Preconditions:** Live wallpaper active on primary and secondary displays.
- **Triggering Event:** User clicks "Exit" from system tray icon context menu or sends `SIGTERM`.
- **Expected Observable Behavior:** Core daemon instructs child players to terminate, cleanly destroys child `HWND`s, invokes `SystemParametersInfoW(SPI_SETDESKWALLPAPER)`, and restores Windows static background within ≤300ms.
- **Verification Method:** Visual frame capture + Win32 desktop handle audit post-termination.
- **Required Evidence:** Video recording showing smooth transition back to static wallpaper; PowerShell check confirming 0 KinetiWall PIDs remaining.
- **Pass Condition:** Desktop static wallpaper fully restored; zero black screen artifacts; zero lingering child windows; zero lingering processes.
- **Fail Condition:** Desktop left black; original wallpaper not restored; zombie processes remain.
- **Severity:** `CRITICAL`
- **Configurations:** All supported Windows releases; x86-64.
- **Dependencies:** `KW-F-008`, Win32 `SystemParametersInfoW`.
- **Status:** `NOT RUN (PLANNED)`
- **Authority:** `USER-REQUIRED`

---

## 3. Domain 2: Core Controller & Process Lifecycle Subsystem

### `KW-AC-005`: Headless Pure-Rust Core Daemon Footprint
- **Requirement ID:** `KW-F-005`, `KW-NF-001`, `KW-NF-002`
- **Title:** Idle Core Controller Memory and CPU Verification
- **Preconditions:** `kinetiwall-core.exe` running in background; Gallery UI closed; single video wallpaper playing.
- **Triggering Event:** Continuous 1-hour idle execution under normal desktop usage.
- **Expected Observable Behavior:** Core controller maintains a private working set between 8 and 12 MB (hard ceiling: ≤15 MB) and idle CPU consumption ≤0.10% (target: 0.00%).
- **Verification Method:** Automated Windows Performance Monitor (PerfMon) / VMMap logging sampled every 5 seconds for 3,600 seconds.
- **Required Evidence:** PerfMon CSV log displaying `Process(kinetiwall-core)\Private Bytes` and `% Processor Time`.
- **Pass Condition:** Peak private working set ≤15.0 MB; average idle CPU ≤0.10%; zero memory creep.
- **Fail Condition:** Private working set >15.0 MB; sustained idle CPU >0.10%; memory linearly drifts upward.
- **Severity:** `CRITICAL`
- **Configurations:** Windows 11 (24H2), Windows 10 (22H2); x86-64.
- **Dependencies:** `KW-F-005`, Rust standard library.
- **Status:** `NOT RUN (PLANNED)`
- **Authority:** `USER-REQUIRED`

### `KW-AC-006`: Ephemeral Gallery UI 0 MB Footprint on Close
- **Requirement ID:** `KW-NF-002`, `KW-NG-001`
- **Title:** Complete Process Termination of Gallery UI on Window Close
- **Preconditions:** KinetiWall core daemon active; wallpaper playing; Gallery UI opened via tray icon.
- **Triggering Event:** User closes the Gallery UI window (clicks 'X' or presses Alt+F4).
- **Expected Observable Behavior:** `kinetiwall-ui.exe` process invokes `std::process::exit(0)` and completely terminates within ≤200ms. 100% of its 70–120 MB RAM footprint is returned to the OS.
- **Verification Method:** Automated test launching UI, sending `WM_CLOSE`, and querying process table for `kinetiwall-ui.exe`.
- **Required Evidence:** Process Explorer / PowerShell log showing `kinetiwall-ui.exe` PID ceases to exist within 200ms; active wallpapers continue playing uninterrupted.
- **Pass Condition:** UI process completely vanishes from Task Manager; 0 MB resident RAM; wallpaper playback uninterrupted.
- **Fail Condition:** UI process remains in Task Manager as a hidden background tray process; UI RAM remains allocated.
- **Severity:** `BLOCKER`
- **Configurations:** All supported Windows releases; x86-64.
- **Dependencies:** Tauri v2, `KW-NG-001`.
- **Status:** `NOT RUN (PLANNED)`
- **Authority:** `USER-REQUIRED`

### `KW-AC-007`: Player Process Isolation & Job Object Orphan Cleanup
- **Requirement ID:** `KW-F-006`, `KW-NF-010`
- **Title:** Child Process Supervision and Automated Orphan Termination
- **Preconditions:** Core daemon running with 2 active wallpaper player child processes (`kinetiwall-player-video.exe`, `kinetiwall-player-web.exe`) bound to Windows Job Object.
- **Triggering Event:** Abrupt termination of `kinetiwall-core.exe` via `taskkill /F /PID {core_pid}`.
- **Expected Observable Behavior:** Windows kernel terminates all child player processes instantaneously due to `JOB_OBJECT_LIMIT_KILL_ON_JOB_CLOSE`.
- **Verification Method:** Automated fault-injection test executing kill command and querying child PIDs within 100ms.
- **Required Evidence:** PowerShell test log verifying 0 orphan player processes remain in process table.
- **Pass Condition:** 100% of child player processes terminated within 100ms; zero orphan processes.
- **Fail Condition:** Any child player process continues running after controller death.
- **Severity:** `CRITICAL`
- **Configurations:** Windows 11, Windows 10; x86-64.
- **Dependencies:** `VG-002`, Win32 Job Objects.
- **Status:** `NOT RUN (PLANNED)`
- **Authority:** `USER-REQUIRED`

### `KW-AC-008`: Bounded Player Crash Auto-Restart
- **Requirement ID:** `KW-F-007`, `KW-NF-009`
- **Title:** Resilient Player Restart with Exponential Backoff
- **Preconditions:** Wallpaper player active on Monitor 1.
- **Triggering Event:** Player process is killed abruptly (simulating an unexpected GPU driver crash).
- **Expected Observable Behavior:** Core daemon detects child exit code, restarts player within ≤1.5s. If player crashes 3 consecutive times within 60 seconds, core halts auto-restart, reverts to static wallpaper, and logs diagnostic error.
- **Verification Method:** Automated script terminating child player PID 1 time (verifying restart) and then 3 times rapidly (verifying backoff & static fallback).
- **Required Evidence:** Engine log traces with timestamps of crash detection, restart attempts, and eventual static fallback.
- **Pass Condition:** Single crash recovers in <1.5s; triple crash halts restart loop and restores static wallpaper cleanly.
- **Fail Condition:** Engine enters infinite crash restart loop; engine crashes when child dies; desktop remains black.
- **Severity:** `MAJOR`
- **Configurations:** All supported Windows platforms; x86-64.
- **Dependencies:** `KW-F-006`, `KW-F-005`.
- **Status:** `NOT RUN (PLANNED)`
- **Authority:** `USER-REQUIRED`

---

## 4. Domain 3: Video Wallpaper Player Subsystem

### `KW-AC-009`: Direct3D 11 Hardware Decoding & CPU Overhead
- **Requirement ID:** `KW-F-009`, `KW-NF-003`, `KW-NF-005`
- **Title:** D3D11VA Hardware Accelerated Video Playback Overhead Verification
- **Preconditions:** Windows 11/10 system with D3D11VA-capable GPU (NVIDIA, AMD, Intel); 4K 60fps HEVC video wallpaper loaded.
- **Triggering Event:** Active wallpaper playback for 600 seconds.
- **Expected Observable Behavior:** Video decodes via GPU hardware decode engine (`hwdec=d3d11va`, `vo=gpu-next`). CPU consumption remains strictly ≤2.5% (target: <1.0% on 1080p).
- **Verification Method:** ETW trace via Windows Performance Recorder (WPR) measuring CPU usage and GPU Video Decode utilization via GPU-Z.
- **Required Evidence:** WPR profile report confirming average CPU <2.5% and GPU 3D engine <3.0%.
- **Pass Condition:** Hardware decoding active; CPU usage ≤2.5%; zero dropped frames during uninterrupted playback.
- **Fail Condition:** CPU decoding fallback (>10% CPU usage); dropped frames >1%; stuttering video.
- **Severity:** `BLOCKER`
- **Configurations:** Windows 11, Windows 10; x86-64 with dedicated or integrated modern GPU.
- **Dependencies:** `VG-003`, `libmpv-2.dll`.
- **Status:** `NOT RUN (PLANNED)`
- **Authority:** `USER-REQUIRED`

### `KW-AC-010`: Zero-Resource Direct3D Swapchain Pause
- **Requirement ID:** `KW-F-010`, `KW-NF-004`
- **Title:** Immediate Rendering Freeze and GPU Load Release on Pause
- **Preconditions:** 4K video wallpaper playing actively.
- **Triggering Event:** Pause command sent via IPC or triggered by fullscreen application detection.
- **Expected Observable Behavior:** Player halts D3D11 swapchain presentation within ≤50ms. Last decoded frame remains frozen on desktop. GPU load attributable to player drops to exactly 0.00%.
- **Verification Method:** High-frequency GPU engine polling via NVAPI / AMD ADL / DXGI performance queries during pause transition.
- **Required Evidence:** Performance trace showing GPU utilization drop to 0.00% within 50ms of pause trigger.
- **Pass Condition:** GPU load = 0.00%; CPU load = 0.00%; last frame frozen without black flicker.
- **Fail Condition:** Player continues rendering in background; GPU usage remains >0.5%; screen turns black.
- **Severity:** `BLOCKER`
- **Configurations:** All supported GPUs; x86-64.
- **Dependencies:** `KW-F-009`, `KW-F-026`.
- **Status:** `NOT RUN (PLANNED)`
- **Authority:** `USER-REQUIRED`

### `KW-AC-011`: Sub-16.6ms Gapless Video Looping
- **Requirement ID:** `KW-F-011`
- **Title:** Seamless Infinite Video Looping Verification
- **Preconditions:** Seamlessly looped 1080p 60fps MP4 video file loaded.
- **Triggering Event:** Playback reaches end-of-file (EOF) boundary.
- **Expected Observable Behavior:** Player loops seamlessly to frame 0 via `loop-file=inf` and internal pre-roll buffer. Presentation delta between final frame and initial frame is ≤16.6ms.
- **Verification Method:** Frame-accurate screen capture / 120 FPS high-speed camera analysis at loop point + PresentMon frame-time log.
- **Required Evidence:** PresentMon CSV log showing frame-time at loop boundary is ≤16.6ms with zero frame drops or audio stutter.
- **Pass Condition:** Delta between EOF and frame 0 ≤16.6ms; zero black frame flash; zero audio click/pop.
- **Fail Condition:** Black frame visible at loop boundary; frame time spike >33.3ms; audio glitches.
- **Severity:** `CRITICAL`
- **Configurations:** Windows 11, Windows 10; x86-64.
- **Dependencies:** `VG-003`, `libmpv-2.dll`.
- **Status:** `NOT RUN (PLANNED)`
- **Authority:** `USER-REQUIRED`

---

## 5. Domain 4: Interactive Web & WebGL Player Subsystem

### `KW-AC-012`: WebView2 Sandboxed Virtual Host Execution
- **Requirement ID:** `KW-F-014`, `KW-F-016`, `KW-NF-016`
- **Title:** Interactive WebGL Wallpaper Hosting with Strict Origin Sandboxing
- **Preconditions:** Microsoft Edge WebView2 Evergreen Runtime installed; complex WebGL 2.0 fluid simulation wallpaper loaded.
- **Triggering Event:** Web player launches and navigates to `https://kinetiwall.local/index.html`.
- **Expected Observable Behavior:** WebGL canvas renders at smooth 60 FPS. Mapped directory isolation prevents reading files outside wallpaper directory. Network requests to external URLs fail with CORS error.
- **Verification Method:** Automated test injecting JavaScript fetch commands: (1) `fetch('https://google.com')`, (2) `fetch('file:///C:/Windows/win.ini')`.
- **Required Evidence:** WebView2 console log confirming both fetch requests rejected with security errors; DXGI present log showing 60 FPS rendering.
- **Pass Condition:** 60 FPS WebGL rendering; external network requests blocked; local filesystem traversal blocked.
- **Fail Condition:** External network request succeeds; file outside directory is read; WebGL fails to render.
- **Severity:** `BLOCKER`
- **Configurations:** All supported Windows versions with WebView2; x86-64.
- **Dependencies:** `VG-006`, Microsoft WebView2 SDK.
- **Status:** `NOT RUN (PLANNED)`
- **Authority:** `USER-REQUIRED`

### `KW-AC-013`: Dynamic Property Injection Bridge
- **Requirement ID:** `KW-F-015`, `KW-F-036`
- **Title:** Bidirectional Property Communication Bridge
- **Preconditions:** Web wallpaper with custom color picker property loaded.
- **Triggering Event:** User modifies color property slider in Gallery UI.
- **Expected Observable Behavior:** Core daemon routes property update over IPC to web player; web player invokes `window.chrome.webview.postMessage`; wallpaper JavaScript receives event and updates canvas uniform in ≤50ms.
- **Verification Method:** Automated integration test injecting property change and asserting DOM / canvas uniform state.
- **Required Evidence:** Browser console log confirming message receipt and execution timestamp <50ms.
- **Pass Condition:** Visual property updates in real-time (<50ms); zero page reload required.
- **Fail Condition:** Property update ignored; requires full page reload; latency >100ms.
- **Severity:** `MAJOR`
- **Configurations:** All supported platforms; x86-64.
- **Dependencies:** `KW-F-014`, Named Pipe IPC.
- **Status:** `NOT RUN (PLANNED)`
- **Authority:** `USER-REQUIRED`

---

## 6. Domain 5: Multi-Monitor Topology & Display Management

### `KW-AC-014`: Independent Multi-Monitor Wallpaper Assignment
- **Requirement ID:** `KW-F-020`, `KW-F-021`
- **Title:** Isolated Multi-Process Wallpaper Presentation across Dual Displays
- **Preconditions:** System with 2 physical monitors connected (Display 1: 2560x1440, Display 2: 1920x1080).
- **Triggering Event:** User assigns video wallpaper to Display 1 and WebGL wallpaper to Display 2.
- **Expected Observable Behavior:** Core daemon spawns 2 distinct child player processes; each window is anchored to the exact coordinates of its respective monitor.
- **Verification Method:** Win32 `GetWindowRect` audit comparing player window rects to `GetMonitorInfoW` coordinates.
- **Required Evidence:** Automated log confirming exact pixel-coordinate match for both displays; independent playback controls verified.
- **Pass Condition:** Display 1 renders video; Display 2 renders WebGL; window boundaries match monitor rects perfectly.
- **Fail Condition:** Wallpaper overlaps between monitors; wrong wallpaper on wrong display; engine crash.
- **Severity:** `CRITICAL`
- **Configurations:** Multi-monitor Windows 11 / Windows 10; x86-64.
- **Dependencies:** `KW-F-006`, `KW-F-020`.
- **Status:** `NOT RUN (PLANNED)`
- **Authority:** `USER-REQUIRED`

### `KW-AC-015`: Mixed-DPI & Display Topology Change Recovery
- **Requirement ID:** `KW-F-024`, `KW-NF-023`
- **Title:** Dynamic Re-Scaling and Placement on Resolution / DPI Changes
- **Preconditions:** Dual-monitor configuration with Display 1 at 150% DPI scale and Display 2 at 100% DPI scale.
- **Triggering Event:** User changes Display 1 scaling from 150% to 100% or unplugs Display 2.
- **Expected Observable Behavior:** Core daemon receives `WM_DPICHANGED` / `WM_DISPLAYCHANGE`, recalculates monitor geometry, resizes player viewports, and cleanly migrates or destroys orphaned players within ≤1.0 second.
- **Verification Method:** Automated test triggering display mode change via `ChangeDisplaySettingsExW` and inspecting window rects.
- **Required Evidence:** Engine log showing display change detection and re-anchor completion in <1,000ms.
- **Pass Condition:** Wallpapers re-align to new bounds without blurring, coordinate drift, or crashes.
- **Fail Condition:** Window offset into black void; UI blurred; player crashes on display disconnect.
- **Severity:** `CRITICAL`
- **Configurations:** Windows 11 (24H2), Windows 10 (22H2); x86-64.
- **Dependencies:** `VG-005`, Win32 Per-Monitor V2 DPI.
- **Status:** `NOT RUN (PLANNED)`
- **Authority:** `USER-REQUIRED`

---

## 7. Domain 6: Performance, Power & Occlusion Management

### `KW-AC-016`: Event-Driven Fullscreen Application Pause
- **Requirement ID:** `KW-F-026`, `KW-NF-004`, `KW-NF-008`
- **Title:** Rapid Occlusion Detection and Playback Pause
- **Preconditions:** Video wallpaper active on primary display.
- **Triggering Event:** User launches or switches to an exclusive fullscreen or borderless maximized 3D game.
- **Expected Observable Behavior:** Core daemon receives `EVENT_SYSTEM_FOREGROUND` via `SetWinEventHook`, calculates that foreground window covers display rect, and dispatches pause command over IPC within ≤50ms.
- **Verification Method:** Automated harness launching a fullscreen test window and timestamping the delta to player swapchain halt.
- **Required Evidence:** Event timestamp log confirming pause command dispatched in ≤50ms.
- **Pass Condition:** Playback paused within 50ms; zero CPU/GPU utilization during occlusion.
- **Fail Condition:** Fullscreen application not detected; pause latency >50ms; wallpaper continues playing behind game.
- **Severity:** `BLOCKER`
- **Configurations:** Windows 11, Windows 10; x86-64.
- **Dependencies:** `VG-004`, Win32 Event Hooks.
- **Status:** `NOT RUN (PLANNED)`
- **Authority:** `USER-REQUIRED`

### `KW-AC-017`: Zero Gaming Frame-Time Degradation
- **Requirement ID:** `KW-F-030`, `KW-NF-005`
- **Title:** Verifiable Zero-Impact Performance During Active 3D Gaming
- **Preconditions:** AAA 3D game (e.g., Cyberpunk 2077 or Counter-Strike 2) running at 1440p high settings.
- **Triggering Event:** 5-minute benchmark run comparing (Scenario A) KinetiWall paused vs (Scenario B) KinetiWall completely closed.
- **Expected Observable Behavior:** KinetiWall consumes 0.00% CPU and 0.00% GPU cycles while paused.
- **Verification Method:** CapFrameX frame-time logging measuring average FPS and 99th percentile (1% low) frame-time across 5 runs per scenario.
- **Required Evidence:** CapFrameX benchmark comparison report.
- **Pass Condition:** Delta in 99th percentile frame-time between Scenario A and Scenario B is strictly <0.5%.
- **Fail Condition:** Measurable FPS drop (>1.0%); frame stuttering or micro-freezes caused by background polling.
- **Severity:** `BLOCKER`
- **Configurations:** Windows 11 x64; dedicated gaming GPU.
- **Dependencies:** `KW-AC-016`, CapFrameX.
- **Status:** `NOT RUN (PLANNED)`
- **Authority:** `USER-REQUIRED`

### `KW-AC-018`: Battery-Saver Power State Transition
- **Requirement ID:** `KW-F-029`, `KW-F-031`
- **Title:** Automatic Playback Pause on AC Power Disconnection
- **Preconditions:** Laptop running Windows 11 on AC power with active live wallpaper; user policy set to "Pause on Battery".
- **Triggering Event:** AC power cord unplugged (simulated via Win32 power broadcast or test harness).
- **Expected Observable Behavior:** Core daemon receives `PBT_APMPOWERSTATUSCHANGE`, verifies `ACLineStatus == 0`, and pauses playback or falls back to static wallpaper within ≤1.0 second.
- **Verification Method:** Automated test broadcasting simulated power status change and verifying player pause state.
- **Required Evidence:** Power transition event log confirming pause execution within 1,000ms.
- **Pass Condition:** Playback halts immediately upon battery transition; resumes when AC power is reconnected.
- **Fail Condition:** Wallpaper continues consuming high power on battery; failure to detect power change.
- **Severity:** `MAJOR`
- **Configurations:** Windows 11 / 10 mobile laptops; x86-64 & ARM64.
- **Dependencies:** Win32 `GetSystemPowerStatus`.
- **Status:** `NOT RUN (PLANNED)`
- **Authority:** `USER-REQUIRED`

---

## 8. Domain 7: Local Wallpaper Library & Customization

### `KW-AC-019`: Manifest Schema V1 Validation & Zip-Slip Protection
- **Requirement ID:** `KW-F-033`, `KW-F-035`, `KW-NF-014`
- **Title:** Secure Manifest Parsing and Archive Import Sanitization
- **Preconditions:** Gallery UI open; user imports a `.kwall` / `.zip` packaged wallpaper archive.
- **Triggering Event:** Malicious archive containing a Zip Slip payload (`../../Windows/malicious.dll`) and malformed `project.json` is imported.
- **Expected Observable Behavior:** Import engine sanitizes all file paths, rejects entries containing relative traversal components (`../`), validates JSON against Serde schema, and displays actionable error message in UI.
- **Verification Method:** Automated unit and integration test importing synthetic malicious archive fixture.
- **Required Evidence:** Test execution log confirming file extraction rejected with `SECURITY_PATH_TRAVERSAL` error.
- **Pass Condition:** Zero files written outside target wallpaper directory; engine remains stable; user notified.
- **Fail Condition:** File extracted outside destination folder; engine crashes; corrupted library.
- **Severity:** `BLOCKER`
- **Configurations:** All supported Windows platforms; x86-64.
- **Dependencies:** Rust `serde_json`, `zip` crate.
- **Status:** `NOT RUN (PLANNED)`
- **Authority:** `USER-REQUIRED`

### `KW-AC-020`: Transactional Configuration Persistence
- **Requirement ID:** `KW-F-037`
- **Title:** Atomic Configuration Persistence and Corruption Recovery
- **Preconditions:** Active wallpaper profiles configured in `config.json`.
- **Triggering Event:** User updates configuration while power loss / abrupt kill is simulated during file write.
- **Expected Observable Behavior:** Configuration writes to a temporary file (`config.json.tmp`) and atomically renames to `config.json` while maintaining `config.json.bak`. If primary file is corrupted, engine auto-restores from backup.
- **Verification Method:** Test harness intentionally injecting truncated bytes into `config.json` and launching engine.
- **Required Evidence:** Engine log confirming automated recovery from `.bak` file with zero data loss.
- **Pass Condition:** Engine restores valid configuration from backup; zero crash on corrupt JSON.
- **Fail Condition:** Engine crashes on startup; user configuration wiped without recovery.
- **Severity:** `CRITICAL`
- **Configurations:** All supported platforms; x86-64.
- **Dependencies:** Rust `std::fs`.
- **Status:** `NOT RUN (PLANNED)`
- **Authority:** `USER-REQUIRED`

---

## 9. Domain 8: WASAPI Audio Reactive Visualization

### `KW-AC-021`: Transient WASAPI Loopback Capture & 60 FPS FFT
- **Requirement ID:** `KW-F-039`, `KW-F-040`, `KW-NF-014`, `KW-NF-015`
- **Title:** Volatile Audio Frequency Capture with Zero Disk Recording
- **Preconditions:** System audio playing (e.g. music stream); audio-reactive web wallpaper active.
- **Triggering Event:** User enables audio reactivity in wallpaper settings.
- **Expected Observable Behavior:** WASAPI loopback captures PCM stream, pure-Rust SIMD calculates 1024-point FFT into 64 frequency bands at 60 FPS, streaming to wallpaper via shared memory (<20ms latency). Audio data exists strictly in volatile memory.
- **Verification Method:** Process Monitor file write audit + 1 kHz test sine wave frequency peak assertion.
- **Required Evidence:** ProcMon trace confirming 0 bytes written to disk; FFT output peak matching 1 kHz bin precisely.
- **Pass Condition:** FFT synchronized with audio (<20ms); CPU overhead <0.30%; zero disk writes; zero network packets.
- **Fail Condition:** Audio audible stutter; FFT desynchronized; audio data written to disk or transmitted over network.
- **Severity:** `CRITICAL`
- **Configurations:** Windows 11, Windows 10; x86-64.
- **Dependencies:** `VG-007`, WASAPI `IAudioClient`.
- **Status:** `NOT RUN (PLANNED)`
- **Authority:** `USER-REQUIRED`

---

## 10. Domain 9: Packaging, Offline Installation & Licensing

### `KW-AC-022`: Air-Gapped Offline Installation & Zero Telemetry
- **Requirement ID:** `KW-F-043`, `KW-NF-006`, `KW-NF-017`, `KW-NF-019`
- **Title:** Completely Offline Setup Execution and Telemetry Audit
- **Preconditions:** Clean Windows 11 virtual machine completely disconnected from network (air-gapped VM).
- **Triggering Event:** Execute `KinetiWall-Setup-v1.0.0-x64.exe` installer package.
- **Expected Observable Behavior:** Installer completes installation, verifies local runtimes, creates shortcuts, and launches engine without requesting internet access. WireShark packet audit confirms zero outbound network packets.
- **Verification Method:** Automated clean VM provisioning, offline installation run, and 48-hour continuous packet capture.
- **Required Evidence:** Installer exit code 0; WireShark log showing exactly 0 packets emitted by KinetiWall executables.
- **Pass Condition:** 100% offline installation; zero cloud account prompts; exactly 0 telemetry packets emitted.
- **Fail Condition:** Installer halts demanding internet connection; analytics packets detected; account sign-in required.
- **Severity:** `BLOCKER`
- **Configurations:** Air-gapped Windows 11 24H2 and Windows 10 22H2 clean installs.
- **Dependencies:** `VG-008`, NSIS/WiX.
- **Status:** `NOT RUN (PLANNED)`
- **Authority:** `USER-REQUIRED`

### `KW-AC-023`: LGPL v2.1+ Dynamic Linkage & DLL Replacement
- **Requirement ID:** `KW-F-046`, `DEC-PLT-004`
- **Title:** Legal Dynamic Linkage Compliance and Library Swapping
- **Preconditions:** Installed KinetiWall application directory containing `libmpv-2.dll`.
- **Triggering Event:** User replaces shipping `libmpv-2.dll` with an independently compiled custom ABI-compatible debug build.
- **Expected Observable Behavior:** KinetiWall launches, dynamically binds to replacement DLL via standard Win32 loader, and renders video successfully without recompilation.
- **Verification Method:** PE header audit via `dumpbin /IMPORTS` + custom DLL replacement execution test.
- **Required Evidence:** `dumpbin` output confirming dynamic import; log file verifying custom debug build signature loaded.
- **Pass Condition:** Binary uses dynamic linkage; user DLL replacement functions flawlessly; license notices bundled.
- **Fail Condition:** Static linking detected (violates LGPL); application crashes on ABI-compatible custom DLL.
- **Severity:** `BLOCKER`
- **Configurations:** All supported Windows platforms; x86-64.
- **Dependencies:** `VG-010`, `DEC-PLT-004`.
- **Status:** `NOT RUN (PLANNED)`
- **Authority:** `APPROVED-DECISION`

