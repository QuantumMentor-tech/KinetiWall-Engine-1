# KinetiWall — Process Responsibility Matrix
**Document ID:** `KW-PRC-002`  
**Deliverable:** Deliverable B — Process Responsibility Matrix  
**Phase:** 2 — System Architecture & Technical Decisions  
**Step:** 8 of 90 — Multi-Process Boundaries & Component Responsibilities  
**Status:** COMPLETE (Authoritative Process Matrix Baseline)  
**Authority:** Principal Software Architect & IPC Systems Lead  
**Date:** 2026-11-10  

---

## 1. Process Identification & Hierarchy

The table below catalogs every process in KinetiWall, detailing its 21 systems engineering attributes across lifecycle, security, resource ownership, and failure handling:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    KINETIWALL PROCESS RESPONSIBILITY MATRIX                 │
├──────────┬─────────────────────────────┬───────────┬────────────────────────┤
│ Proc ID  │ Process Executable Name     │ Status    │ Process Role & Scope   │
├──────────┼─────────────────────────────┼───────────┼────────────────────────┤
│ PROC-01  │ kinetiwall-core.exe         │ APPROVED  │ Core Controller Daemon │
│ PROC-02  │ kinetiwall-ui.exe           │ APPROVED  │ Ephemeral Gallery UI   │
│ PROC-03  │ kinetiwall-player-video.exe │ APPROVED  │ Isolated Video Player  │
│ PROC-04  │ kinetiwall-player-web.exe   │ APPROVED  │ Isolated Web Player    │
│ PROC-05  │ kinetiwall-player-native.exe│ DEFERRED  │ Native 3D Host (V2.0+) │
└──────────┴─────────────────────────────┴───────────┴────────────────────────┘
```

---

## 2. Exhaustive 21-Attribute Process Specifications

### `PROC-01`: Core Controller Daemon (`kinetiwall-core.exe`)
1. **Process ID:** `PROC-01`
2. **Proposed Process Name:** `kinetiwall-core.exe`
3. **Approval Status:** `APPROVED` (`ADR-0001`, `ADR-0002`)
4. **Primary Purpose:** Authoritative background session supervisor, display manager, occlusion monitor, Job Object creator, and Named Pipe IPC server.
5. **Language & Runtime:** Pure Rust (2021 Edition, MSVC native runtime).
6. **Launch Owner:** Windows Shell via user logon startup registry (`HKCU\...\Run`) or manual user shortcut.
7. **Execution Context:** Interactive User Session (Session ID ≥ 1, Medium Integrity Level). Not a Windows Service.
8. **Required Permissions:** Standard user permissions. Zero UAC elevation required.
9. **Authoritative State Owned:** Global engine lifecycle state, active display geometry map, per-monitor wallpaper assignments, occlusion pause flags, and persistent configuration.
10. **Resources Owned:** Win32 system tray icon, Named Pipe server handle, Windows Job Object handle, single-instance mutex handle.
11. **Resources Accessed:** `%LOCALAPPDATA%\KinetiWall\config.json`, `%LOCALAPPDATA%\KinetiWall\wallpapers\`.
12. **Commands Accepted:** `ApplyWallpaper`, `SetGlobalPause`, `UpdateProperty`, `QueryState`, `PlayerReady`, `PlaybackError`.
13. **Events Emitted:** `StateChanged`, `DisplayTopologyChanged`, `PowerSourceChanged`, `ShutdownInitiated`.
14. **External Dependencies:** Windows User32 (`Progman`, `WorkerW`), Windows Core Audio (WASAPI), Kernel32 Power APIs.
15. **Startup Prerequisites:** Interactive user session initialized; single-instance mutex acquired.
16. **Shutdown Behavior:** Signals child players to terminate, detaches child `HWND`s, terminates Job Object, restores static wallpaper, and exits in ≤300ms.
17. **Crash Behavior:** Global panic hook restores static wallpaper; closing the Job Object handle causes the kernel to terminate all child players automatically (`KILL_ON_JOB_CLOSE`).
18. **Restart Policy:** Started on next user logon; critical fatal crash logged to `crash.log`.
19. **Logging Responsibility:** Core daemon events, IPC transactions, and crash dumps written to `%LOCALAPPDATA%\KinetiWall\logs\core.log`.
20. **Resource Limits / Performance Expectations:** Private working set: **8–12 MB RAM** (ceiling: ≤15 MB); idle CPU: **≤0.10%** (target: 0.00%).
21. **Requirements Satisfied:** `KW-F-001`..`005`, `KW-F-008`, `KW-F-020`..`027`, `KW-NF-001`..`004`, `KW-NG-001`..`004`.

---

### `PROC-02`: Ephemeral Gallery UI (`kinetiwall-ui.exe`)
1. **Process ID:** `PROC-02`
2. **Proposed Process Name:** `kinetiwall-ui.exe`
3. **Approval Status:** `APPROVED` (`ADR-0004`)
4. **Primary Purpose:** On-demand desktop interface for browsing wallpapers, adjusting dynamic properties, importing archives, and managing settings.
5. **Language & Runtime:** Tauri v2 (Rust backend host) + Web Frontend (TypeScript, Svelte/React, Tailwind CSS).
6. **Launch Owner:** `kinetiwall-core.exe` (via system tray context menu or user shortcut).
7. **Execution Context:** Interactive User Session (Session ID ≥ 1, Medium Integrity Level).
8. **Required Permissions:** Standard user permissions. Zero UAC elevation required.
9. **Authoritative State Owned:** UI view state (active tab, scroll position, search query filter, modal dialog state).
10. **Resources Owned:** Gallery window `HWND`, Tauri webview environment instance.
11. **Resources Accessed:** `%LOCALAPPDATA%\KinetiWall\cache\thumbnails\`, Named Pipe IPC client stream.
12. **Commands Accepted:** `OpenWindow`, `FocusWindow`, `CloseWindow`.
13. **Events Emitted:** `ApplyWallpaperRequest`, `UpdatePropertyRequest`, `SetGlobalPauseRequest`, `ImportArchiveRequest`.
14. **External Dependencies:** Microsoft Edge WebView2 runtime (via Tauri).
15. **Startup Prerequisites:** `kinetiwall-core.exe` must be running and listening on local Named Pipe.
16. **Shutdown Behavior:** Window close (`WM_CLOSE`) executes `std::process::exit(0)`, completely terminating process and freeing 100% of memory.
17. **Crash Behavior:** Abrupt UI crash terminates UI process cleanly without affecting core daemon or active wallpaper playback.
18. **Restart Policy:** Zero auto-restart. User manually launches UI from tray when needed.
19. **Logging Responsibility:** UI console logs and frontend errors written to `%LOCALAPPDATA%\KinetiWall\logs\ui.log`.
20. **Resource Limits / Performance Expectations:** Active memory: 70–120 MB RAM; **Closed memory: exactly 0.0 MB RAM** (`KW-NG-001`).
21. **Requirements Satisfied:** `KW-F-034`..`036`, `KW-NF-002`, `KW-NF-011`, `KW-NG-001`.

---

### `PROC-03`: Isolated Video Player (`kinetiwall-player-video.exe`)
1. **Process ID:** `PROC-03`
2. **Proposed Process Name:** `kinetiwall-player-video.exe`
3. **Approval Status:** `APPROVED` (`ADR-0003`, `ADR-0006`)
4. **Primary Purpose:** Out-of-process Direct3D 11 hardware-accelerated video decoding, infinite gapless looping, and presentation.
5. **Language & Runtime:** Pure Rust with dynamically linked `libmpv-2.dll` (LGPL v2.1+).
6. **Launch Owner:** `kinetiwall-core.exe` (via `ProcessOrchestrator`).
7. **Execution Context:** Interactive User Session (Session ID ≥ 1, Medium Integrity Level, Job Object sandbox).
8. **Required Permissions:** Standard user permissions. Zero UAC elevation required.
9. **Authoritative State Owned:** Internal video playback position, frame presentation clock, volume level, D3D11 swapchain backbuffer.
10. **Resources Owned:** Direct3D 11 device, DXGI swapchain, child rendering window `HWND`, `mpv_handle`.
11. **Resources Accessed:** User video file path (`.mp4`, `.mkv`, `.webm`).
12. **Commands Accepted:** `Play(path, loop)`, `Pause`, `Resume`, `Stop`, `SetVolume(val)`, `Terminate`.
13. **Events Emitted:** `PlayerReady(HWND)`, `PlaybackStarted`, `LoopBoundaryReached`, `PlaybackError`, `ProcessTerminating`.
14. **External Dependencies:** Direct3D 11 GPU driver, `libmpv-2.dll`, Windows DWM.
15. **Startup Prerequisites:** Valid display ID and IPC pipe handle passed via command-line arguments.
16. **Shutdown Behavior:** Halts present loop, releases DXGI swapchain, destroys window `HWND`, and exits cleanly.
17. **Crash Behavior:** Process death detected by core daemon via signaled process handle; triggers exponential backoff restart.
18. **Restart Policy:** Bounded auto-restart: max 3 attempts within 60 seconds; static wallpaper fallback if exceeded.
19. **Logging Responsibility:** libmpv C-API log callbacks redirected to `%LOCALAPPDATA%\KinetiWall\logs\player_video_{display}.log`.
20. **Resource Limits / Performance Expectations:** Private RAM: 45–90 MB; 4K CPU: **≤2.50%**; Paused GPU: **0.00%**; Loop delta: **≤16.6ms**.
21. **Requirements Satisfied:** `KW-F-009`..`013`, `KW-NF-003`, `KW-NF-005`, `KW-NF-006`.

---

### `PROC-04`: Isolated Web Player (`kinetiwall-player-web.exe`)
1. **Process ID:** `PROC-04`
2. **Proposed Process Name:** `kinetiwall-player-web.exe`
3. **Approval Status:** `APPROVED` (`ADR-0003`, `ADR-0007`, `ADR-0013`)
4. **Primary Purpose:** Out-of-process container hosting interactive HTML5, Canvas 2D, WebGL 1.0/2.0, and CSS3 wallpapers under strict sandboxing.
5. **Language & Runtime:** Pure Rust host embedding Microsoft Edge WebView2 Evergreen Runtime.
6. **Launch Owner:** `kinetiwall-core.exe` (via `ProcessOrchestrator`).
7. **Execution Context:** Interactive User Session (Session ID ≥ 1, Medium Integrity Level, Job Object sandbox with 500 MB RAM limit).
8. **Required Permissions:** Standard user permissions. Zero UAC elevation required.
9. **Authoritative State Owned:** DOM execution context, WebGL rendering context, custom dynamic property state.
10. **Resources Owned:** Child rendering `HWND`, WebView2 environment controller, shared memory audio reader.
11. **Resources Accessed:** Mapped local folder `https://kinetiwall.local/` (read-only).
12. **Commands Accepted:** `Navigate(url)`, `UpdateProperty(json)`, `ForwardInput(event)`, `Pause`, `Resume`, `Terminate`.
13. **Events Emitted:** `PlayerReady(HWND)`, `WebMessageReceived`, `ScriptError`, `ProcessTerminating`.
14. **External Dependencies:** Microsoft Edge WebView2 runtime, Direct3D 11 GPU driver.
15. **Startup Prerequisites:** WebView2 runtime installed; valid display ID and IPC pipe handle passed.
16. **Shutdown Behavior:** Closes WebView2 controller, releases rendering surface, and terminates process.
17. **Crash Behavior:** Job Object terminates process if 500 MB breached; core daemon detects death and logs error.
18. **Restart Policy:** Auto-restarted up to 3 times for normal crashes; zero auto-restart if terminated by 500 MB memory quota breach.
19. **Logging Responsibility:** WebView2 browser console logs routed to `%LOCALAPPDATA%\KinetiWall\logs\player_web_{display}.log`.
20. **Resource Limits / Performance Expectations:** Private RAM: 120–250 MB (hard ceiling: **500 MB**); WebGL frame rate: **60 FPS**.
21. **Requirements Satisfied:** `KW-F-014`..`018`, `KW-NF-007`, `KW-NF-016`, `KW-NG-006`.

---

### `PROC-05`: Native 3D Executable Host (`kinetiwall-player-native.exe` — Post-V1 Candidate)
1. **Process ID:** `PROC-05`
2. **Proposed Process Name:** `kinetiwall-player-native.exe`
3. **Approval Status:** `DEFERRED (V2.0+ HORIZON)` (`ADR-0014`, `V1-EXCL-009`)
4. **Primary Purpose:** Supervised process container for reparenting compiled standalone 3D game executables (Godot/Unity .exe).
5. **Language & Runtime:** Pure Rust Win32 window-reparenting supervisor.
6. **Launch Owner:** `kinetiwall-core.exe`.
7. **Execution Context:** Restricted Token AppContainer / Job Object sandbox.
8. **Required Permissions:** Standard user permissions.
9. **Authoritative State Owned:** External process handle, injected window style masks (`WS_CHILD | WS_CLIPSIBLINGS`).
10. **Resources Owned:** Child reparenting container `HWND`, input trap hooks.
11. **Resources Accessed:** Standalone game binary path.
12. **Commands Accepted:** `Launch(exe_path)`, `Pause`, `Resume`, `Terminate`.
13. **Events Emitted:** `HostReady(HWND)`, `HostError`, `ProcessExited`.
14. **External Dependencies:** Win32 `SetParent`, Direct3D 11/12, Vulkan drivers.
15. **Startup Prerequisites:** Formal approval of Prototype Spike `VG-009` proving zero input focus stealing.
16. **Shutdown Behavior:** Terminate external executable process gracefully via `WM_CLOSE`, then hard terminate via Job Object.
17. **Crash Behavior:** Process exit caught by supervisor; does not impact core daemon.
18. **Restart Policy:** Zero auto-restart (external executables may crash persistently).
19. **Logging Responsibility:** Standard stdout/stderr capture written to `player_native_{display}.log`.
20. **Resource Limits / Performance Expectations:** Unresolved pending `VG-009` prototype benchmark.
21. **Requirements Satisfied:** Post-V1 roadmap enhancement candidate.

