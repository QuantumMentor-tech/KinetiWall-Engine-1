# KinetiWall — Process Topology Specification
**Document ID:** `KW-PRC-001`  
**Deliverable:** Deliverable A — Process Topology Specification  
**Phase:** 2 — System Architecture & Technical Decisions  
**Step:** 8 of 90 — Multi-Process Boundaries & Component Responsibilities  
**Status:** COMPLETE (Authoritative Process Topology Baseline)  
**Authority:** Principal Software Architect & IPC Systems Lead  
**Date:** 2026-11-10  

---

## 1. Process Topology Architecture

KinetiWall organizes its runtime execution into a **Strictly Bounded Triadic Multi-Process Architecture** running in the interactive Windows user session (Session ID ≥ 1).

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    KINETIWALL RUNTIME PROCESS TOPOLOGY                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   ┌──────────────────────────────────────────────────────────────────────┐  │
│   │ [PROC-01] CORE CONTROLLER DAEMON: kinetiwall-core.exe                │  │
│   │  • Language: Pure Rust                                               │  │
│   │  • Execution Context: User Interactive Session (Session ID >= 1)     │  │
│   │  • Memory Budget: 8–12 MB private working set (hard ceiling <=15 MB) │  │
│   │  • Responsibilities: State authority, display discovery, occlusion  │  │
│   │    monitoring, Job Object supervisor, Named Pipe IPC server.         │  │
│   └───────────────────┬───────────────────────────────┬──────────────────┘  │
│                       │ Local Named Pipe IPC          │ Windows Job Object  │
│                       ▼                               ▼ (KillOnJobClose)    │
│   ┌───────────────────────────────┐ ┌────────────────────────────────────┐  │
│   │ [PROC-02] EPHEMERAL GALLERY UI│ │ [PROC-03] ISOLATED VIDEO PLAYER    │  │
│   │  kinetiwall-ui.exe (Tauri v2) │ │  kinetiwall-player-video.exe       │  │
│   │  • Lifecycle: On-demand       │ │  • Engine: libmpv-2.dll (LGPL v2.1)│  │
│   │  • Closed Memory: Exactly 0 MB│ │  • Graphics: Direct3D 11VA HW      │  │
│   │  • Action: std::process::exit │ │  • Looping: loop-file=inf gapless  │  │
│   │    on window close            │ │  • Memory: 45–90 MB private RAM    │  │
│   └───────────────────────────────┘ └────────────────────────────────────┘  │
│                                                       │                     │
│                                                       │ Windows Job Object  │
│                                                       ▼ (500 MB Limit)      │
│                                     ┌────────────────────────────────────┐  │
│                                     │ [PROC-04] ISOLATED WEB PLAYER      │  │
│                                     │  kinetiwall-player-web.exe         │  │
│                                     │  • Engine: Microsoft Edge WebView2 │  │
│   [POST-V1 CANDIDATE - PROC-05]     │  • Sandbox: Virtual Host CORS Deny │  │
│   ┌───────────────────────────────┐ │  • Graphics: WebGL 2.0 @ 60 FPS    │  │
│   │ Native Executable Reparenting │ │  • Memory: 120–250 MB private RAM  │  │
│   │ Container (DEFERRED TO V2.0+) │ └────────────────────────────────────┘  │
│   └───────────────────────────────┘                                         │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Process Boundary Justifications

Every process boundary in KinetiWall is justified by a specific systems engineering requirement:

| Process Identity | Executable Name | Isolation Rationale & Justification | Impact if Collapsed into Core (Anti-Pattern) |
|:---|:---|:---|:---|
| **Core Controller** | `kinetiwall-core.exe` | **State Authority & Shell Stability:** Must run 24/7 as an ultra-lightweight (8–12 MB) daemon. Owns the single source of truth for display geometries, window hooks, and settings. | If UI or media were bundled here, memory would swell to 150+ MB and a media crash would kill the desktop wallpaper engine. |
| **Gallery UI** | `kinetiwall-ui.exe` | **Ephemeral Resource Release (`KW-NG-001`):** User interface operations require heavy web/webview runtimes (70–120 MB RAM). When closed, the process terminates completely to 0 MB RAM. | If kept resident in tray, 70–120 MB RAM is wasted permanently while users are gaming or working. |
| **Video Player** | `kinetiwall-player-video.exe` | **Codec Fault Isolation & GPU Memory Guard:** Media decoding interacts directly with GPU drivers and third-party decoders (`libmpv`/FFmpeg), which can crash on corrupted video streams. | A corrupt MP4 or GPU TDR driver crash would crash the core engine, invalidating the desktop shell. |
| **Web Player** | `kinetiwall-player-web.exe` | **Untrusted Code Sandboxing & Memory Bomb Containment:** Web wallpapers run third-party JavaScript/WebGL. Out-of-process isolation allows hard 500 MB Job Object quotas and origin isolation. | A runaway JavaScript `while(true)` memory allocation loop would crash the core engine and freeze Windows. |

---

## 3. Analysis of Specific Process Models

### 3.1 Core Controller: User Session vs Windows Service
- **Decision:** Interactive User Session Daemon (`Session ID ≥ 1`).
- **Rejected Alternative:** Windows Service (Session 0).
- **Technical Rationale:** Since Windows Vista, Windows Services execute in Session 0. Session 0 is completely isolated from the interactive user desktop. A Session 0 service **cannot interact with desktop HWNDs, cannot send message `0x052C` to `Progman`, cannot reparent windows under `WorkerW`, and cannot receive user tray mouse events**. Attempting cross-session window manipulation via named pipes or user tokens introduces massive attack surfaces and elevation risks. Running as a standard non-elevated user-session daemon (`Medium IL`) completely satisfies all desktop requirements without UAC elevation (`ADR-0002`).

### 3.2 Gallery UI Process: The Tauri v2 Process Architecture
- **Packaging Model:** KinetiWall builds `kinetiwall-ui.exe` as a **dedicated standalone Tauri v2 application** distinct from `kinetiwall-core.exe`.
- **Relationship:**
  - `kinetiwall-core.exe` is the background supervisor.
  - When the user double-clicks the system tray icon, `core` spawns `kinetiwall-ui.exe`.
  - `kinetiwall-ui.exe` initializes its Tauri webview, connects as an IPC client to `\\.\pipe\kinetiwall-ipc-{SessionId}`, fetches state, and renders the gallery interface.
  - When the user closes the window (`WM_CLOSE`), Tauri's `on_window_event` intercepts the close event and invokes `std::process::exit(0)`.
  - The OS reclaims all 70–120 MB RAM immediately. Zero background UI processes remain (`KW-NG-001`).

### 3.3 Video Player Strategy Evaluation

To establish the authoritative architecture, three media-engine integration strategies were evaluated:

| Strategy Dimension | Candidate 1: Embedded `libmpv` (C-API FFI) | Candidate 2: Supervised `mpv.exe` External CLI | Candidate 3: Windows Media Foundation (WMF) |
|:---|:---|:---|:---|
| **Process Model** | Dedicated `kinetiwall-player-video.exe` embedding `libmpv-2.dll` via dynamic link. | Core launches upstream `mpv.exe` via CLI flags (`--wid={HWND}`). | In-process Direct3D 11 WMF player. |
| **Direct3D Control** | **Full Direct Control:** Player creates D3D11 device and swapchain; passes native texture surface to mpv via `vo=gpu-next`. | **Indirect:** `mpv.exe` creates its own hidden swapchain and blits to target HWND. | Native Direct3D 11 surfaces via Media Engine API. |
| **Loop Latency** | **Sub-16.6ms Gapless:** Uses `loop-file=inf` and internal pre-roll buffer in volatile memory. | **Variable Stutter (50–200ms):** CLI process IPC introduces frame delay at boundary. | High loop latency (100–300ms gap) unless complex custom topologies written. |
| **Codec Coverage** | **Universal:** H.264, HEVC, VP9, AV1 in MP4, MKV, WebM out of the box. | **Universal:** Identical FFmpeg decoding engine. | **Limited:** VP9 and AV1 fail unless user installed optional Microsoft Store codecs. |
| **Process Count** | 1 process per active video wallpaper. | 1 process per active video wallpaper. | 0 additional processes (in-process). |
| **Fault Isolation** | **Complete:** Crash in `libmpv` terminates only the player subprocess. | **Complete:** Crash in `mpv.exe` terminates only the external process. | **Zero:** Codec crash terminates the entire engine. |
| **Licensing** | **LGPL v2.1+ Compliant:** Dynamic linking satisfies Section 6 (`DEC-PLT-004`). | GPL / LGPL compliant. | Proprietary Microsoft APIs. |
| **Status** | **`ACCEPTED` (`ADR-0006`)** | `REJECTED` (Higher latency, fragile window embedding) | `REJECTED` (Missing open codecs, no crash isolation) |

### 3.4 Interactive Web Player: WebView2 Sandboxed Container
- **Process Identity:** `kinetiwall-player-web.exe`.
- **Runtime:** Microsoft Edge WebView2 Evergreen Runtime (`WebView2Loader.dll`).
- **Security Boundaries:**
  - Virtual Host Directory Mapping: Mapped to `https://kinetiwall.local/` with `COREWEBVIEW2_HOST_RESOURCE_ACCESS_KIND_DENY_CORS`. All external HTTP/HTTPS network requests fail.
  - Disabled Schemes: `file://` scheme disabled to prevent local disk traversal.
  - Hardware Permissions: Camera, microphone, and geolocation requests are auto-denied without user prompts.
  - Window Popups: `window.open` and protocol navigations (`steam://`, `cmd.exe`) are intercepted and canceled.
  - Resource Guardrails: Bound to Windows Job Object with a hard 500 MB memory commit limit (`KW-F-018`).

### 3.5 Optional Native 3D or Executable Host (`PROC-05`)
- **Status in V1.0:** `PROPOSED (CONFIRMED V1.0 SCOPE RESOLUTION)`
- **V1.0 Decision:** All 3D live wallpapers in Version 1.0 are delivered via **WebGL 2.0 packages (Three.js, Babylon.js, or Godot 4 WebGL export templates) hosted inside the sandboxed Web Player (`PROC-04`)**.
- **Post-V1 Horizon (`V2.0+`):** Standalone compiled Win32 game executable reparenting (`.exe`) is subjected to Phase 2 Prototype Spike `VG-009`. It is formally excluded from V1.0 (`V1-EXCL-009`) due to severe mouse-trapping and focus-stealing hazards.

---

## 4. External Operating System Dependencies

The application relies on 8 external Windows OS subsystems:

| External Dependency | Windows API / Interface | Failure Mode | KinetiWall Recovery Behavior |
|:---|:---|:---|:---|
| **Windows Explorer** | `Progman` / `WorkerW` / `SHELLDLL_DefView` | Shell crash or restart (`explorer.exe` death). | Catches `TaskbarCreated` registered message; re-executes `0x052C` handshake in ≤2.5s (`KW-F-003`). |
| **Display Subsystem** | Win32 `EnumDisplayMonitors` / `WM_DISPLAYCHANGE` | Monitor disconnect, hotplug, resolution change. | Dynamically re-queries display rects; re-anchors surviving viewports; terminates disconnected players (`KW-F-024`). |
| **Power Management** | Win32 `GetSystemPowerStatus` / Power Hooks | AC disconnect; laptop switches to battery. | Pauses playback or falls back to static wallpaper based on user policy (`KW-F-029`). |
| **Windows Core Audio**| WASAPI `IAudioClient` loopback capture | Headphone unplugged (`WM_DEVICECHANGE`). | Automatically re-binds capture client to new default audio endpoint in <500ms (`KW-F-042`). |
| **WebView2 Runtime** | Microsoft Edge Evergreen Runtime | Missing or corrupted Edge runtime. | Pre-flight check detects missing runtime; disables web wallpapers gracefully while video wallpapers work (`FAIL-10`). |
| **Graphics Drivers** | Direct3D 11 / DXGI Swapchains | GPU TDR driver reset (`DXGI_ERROR_DEVICE_RESET`). | Player detects device loss; releases all swapchains; recreates D3D11 device and resumes playback (`FAIL-09`). |
| **Local Filesystem** | `%LOCALAPPDATA%\KinetiWall` storage | Power loss during configuration write. | Atomic file rename ensures primary `config.json` is never half-written; auto-restores from `.bak` (`KW-F-037`). |
| **Offline Installer** | NSIS / WiX standalone executable | Missing internet connection during setup. | Package bundles all pre-compiled engine binaries and runtimes for 100% offline installation (`KW-NG-003`). |

