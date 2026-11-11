# KinetiWall — System Architecture Overview
**Document ID:** `KW-ARC-001`  
**Deliverable:** Deliverable A — System Architecture Overview  
**Phase:** 2 — System Architecture & Technical Decisions  
**Step:** 7 of 90 — System Architecture & Architecture Decision Records  
**Status:** COMPLETE (Authoritative Architectural Baseline)  
**Authority:** Principal Software Architect & Rust Systems Lead  
**Date:** 2026-11-10  

---

## 1. System Context & External Environment

KinetiWall operates as a native user-session application tightly integrated with the Windows shell subsystem, the display hardware infrastructure, and the local multimedia filesystem.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          KINETIWALL SYSTEM CONTEXT                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   ┌───────────────────────┐                 ┌───────────────────────────┐   │
│   │    WINDOWS SHELL      │                 │     HARDWARE / DISPLAY    │   │
│   │  • explorer.exe       │                 │  • Win32 EnumDisplayMon   │   │
│   │  • Progman / WorkerW  │                 │  • Per-Monitor DPI (V2)   │   │
│   │  • SHELLDLL_DefView   │                 │  • D3D11 Graphics Adapter │   │
│   └───────────▲───────────┘                 └─────────────▲─────────────┘   │
│               │ (HWND Attachment)                         │ (Hardware Swap) │
│   ┌───────────┴───────────────────────────────────────────┴─────────────┐   │
│   │                          KINETIWALL ENGINE                          │   │
│   │  ┌─────────────────┐   Local IPC   ┌─────────────────────────────┐  │   │
│   │  │ kinetiwall-core │<─────────────>│ kinetiwall-ui (Ephemeral)   │  │   │
│   │  └────────┬────────┘  (Named Pipe) └─────────────────────────────┘  │   │
│   │           │ (Job Object Supervision)                                │   │
│   │           ▼                                                         │   │
│   │  ┌───────────────────────────────┐ ┌─────────────────────────────┐  │   │
│   │  │ kinetiwall-player-video       │ │ kinetiwall-player-web        │  │   │
│   │  │ (libmpv D3D11VA HW Looping)   │ │ (WebView2 Sandboxed WebGL)  │  │   │
│   │  └───────────────────────────────┘ └─────────────────────────────┘  │   │
│   └───────────┬───────────────────────────────────────────┬─────────────┘   │
│               │ (WASAPI Loopback Capture)                 │ (JSON Manifest) │
│   ┌───────────▼───────────┐                 ┌─────────────▼─────────────┐   │
│   │  WINDOWS CORE AUDIO   │                 │   LOCAL ASSET STORAGE     │   │
│   │  • WASAPI AudioClient │                 │  • %LOCALAPPDATA%\Wallp.  │   │
│   │  • Transient PCM FFT  │                 │  • Declarative Manifests  │   │
│   │  • Volatile Ring Buff │                 │  • Atomic config.json.bak │   │
│   └───────────────────────┘                 └───────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Context Boundary Rules
1. **Interactive User Session Only:** KinetiWall executes strictly within the user's interactive logon session (Session ID ≥ 1). It is **not a Windows Service** (Session 0 isolation prevents GUI window interaction and desktop message broadcast).
2. **Offline-First Storage:** Wallpapers, manifests, thumbnails, and configuration reside 100% on the local filesystem. Zero cloud accounts or external telemetry endpoints are permitted (`KW-NG-002`, `KW-NG-003`).
3. **Hardware Presentation:** Video and WebGL swapchains present directly into win32 child windows reparented beneath `WorkerW` or `Progman`.

---

## 2. Container & Process View

KinetiWall partitions runtime responsibilities into four distinct executable containers to guarantee memory isolation, crash containment, and ephemeral UI resource release:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    KINETIWALL RUNTIME PROCESS TOPOLOGY                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   ┌──────────────────────────────────────────────────────────────────────┐  │
│   │ [1] CORE CONTROLLER DAEMON: kinetiwall-core.exe                      │  │
│   │     • Language: Pure Rust                                            │  │
│   │     • Lifecycle: Continuous session background (tray icon)           │  │
│   │     • Memory Budget: 8–12 MB private working set (hard ceiling ≤15MB)│  │
│   │     • Responsibilities: State authority, display discovery, occlusion│  │
│   │       monitoring, Job Object supervisor, Named Pipe IPC server.      │  │
│   └───────────────────┬───────────────────────────────┬──────────────────┘  │
│                       │ Local Named Pipe IPC          │ Job Object Limits   │
│                       ▼                               ▼                     │
│   ┌───────────────────────────────┐ ┌────────────────────────────────────┐  │
│   │ [2] EPHEMERAL GALLERY UI      │ │ [3] ISOLATED VIDEO PLAYER PROCESS  │  │
│   │     kinetiwall-ui.exe         │ │     kinetiwall-player-video.exe    │  │
│   │     • Tech: Tauri v2 (Rust+TS)│ │     • Engine: libmpv-2.dll (LGPL)  │  │
│   │     • Lifecycle: On-demand    │ │     • Graphics: Direct3D 11VA HW   │  │
│   │     • Closed Footprint: 0 MB  │ │     • Looping: loop-file=inf gapless│ │
│   │     • Action: std::process::  │ │     • Memory: 45–90 MB private RAM │  │
│   │       exit(0) on window close │ │     • Pause: Swapchain freeze (0%) │  │
│   └───────────────────────────────┘ └────────────────────────────────────┘  │
│                                                       │                     │
│                                                       │ Job Object Limits   │
│                                                       ▼                     │
│                                     ┌────────────────────────────────────┐  │
│                                     │ [4] ISOLATED WEB PLAYER PROCESS    │  │
│                                     │     kinetiwall-player-web.exe      │  │
│                                     │     • Engine: WebView2 Evergreen   │  │
│                                     │     • Security: Virtual Host CORS  │  │
│                                     │     • Graphics: WebGL 2.0 @ 60 FPS │  │
│                                     │     • Quota: 500 MB hard Job limit │  │
│                                     └────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Process Architectural Commitments
1. **`kinetiwall-core.exe` (Core Controller):** The single source of truth. Runs headless with a Win32 system tray icon (`Shell_NotifyIconW`). It manages monitor geometries, foreground window hooks, power transitions, child process supervision, and configuration persistence.
2. **`kinetiwall-ui.exe` (Gallery UI):** Built with Tauri v2. It is launched on-demand by the tray icon or shortcut. It operates strictly as an IPC client. **Crucial Governance Invariant (`KW-NG-001`):** Closing the gallery window executes `std::process::exit(0)`, completely terminating the process and freeing 100% of its 70–120 MB RAM footprint.
3. **`kinetiwall-player-video.exe` (Video Player):** Spawned out-of-process for each active video wallpaper. Dynamically links `libmpv-2.dll` (LGPL v2.1+) with Direct3D 11 hardware decoding (`vo=gpu-next`, `hwdec=d3d11va`). Bound to a Windows Job Object (`JOB_OBJECT_LIMIT_KILL_ON_JOB_CLOSE`).
4. **`kinetiwall-player-web.exe` (Web Player):** Spawned out-of-process for interactive HTML5/Canvas/WebGL wallpapers. Hosts Microsoft Edge WebView2 mapped to virtual origin `https://kinetiwall.local/`. Bound to Windows Job Object with a 500 MB memory limit.

---

## 3. Explicit Resource Ownership Model

To eliminate state synchronization hazards, KinetiWall enforces a strict **Single Source of Truth (SSOT)** model across all system resources:

| Resource Dimension | Authoritative Owner | Secondary Observers / Clients | Ownership & Lifecycle Rules |
|:---|:---|:---|:---|
| **Child Player Processes** | `ProcessOrchestrator` (`core`) | Windows Job Object | Spawns, monitors health via exit codes, enforces 3-attempt restart backoff, and terminates child processes. |
| **Desktop Rendering HWNDs** | `DesktopHostManager` (`core`) | Wallpaper Players | Owns parent `WorkerW`/`Progman` handles; assigns child rendering viewports; recovers on Explorer restart. |
| **Monitor-to-Wallpaper Map** | `DisplayTopologyManager` (`core`) | Gallery UI, Players | Maps physical display IDs to active wallpaper paths; reconciles mixed-DPI scaling and display hotplugs. |
| **Playback & Occlusion State** | `ResourcePolicyEngine` (`core`) | Players, Tray Icon | Evaluates fullscreen hooks, battery power, and user toggles; dispatches PAUSE/RESUME/STOP commands. |
| **Named Pipe Server** | `IpcServer` (`core`) | Gallery UI, Players | Hosts `\\.\pipe\kinetiwall-ipc-{SessionId}`; enforces logon SID DACL; validates JSON-RPC schemas. |
| **Wallpaper Manifests & Assets**| `LibraryRepository` (`core`) | Gallery UI | Reads/writes `%LOCALAPPDATA%\KinetiWall\wallpapers\`; parses `project.json`; sanitizes import paths. |
| **Application Configuration** | `ConfigurationRepository` (`core`) | Gallery UI | Owns `config.json`; writes atomically to `.tmp` with rename; maintains `config.json.bak` rollback. |
| **Audio Capture Stream** | `AudioCaptureEngine` (`core`) | Web Wallpaper Players | Captures WASAPI loopback; computes 1024-point FFT in pure Rust; streams arrays over volatile shared memory. |
| **Thumbnail Disk Cache** | `ThumbnailManager` (`core`) | Gallery UI | Generates and caches WebP/PNG preview thumbnails in `%LOCALAPPDATA%\KinetiWall\cache\thumbnails\`. |

---

## 4. Desktop Shell Integration Architecture

Desktop wallpaper presentation is governed by the `DesktopHostManager` component inside `kinetiwall-core.exe`. It isolates the undocumented Windows shell APIs behind a clean, testable Win32 abstraction:

```mermaid
flowchart TD
    START["DesktopHostManager::Initialize()"] --> SEND["SendMessageTimeoutW(Progman, 0x052C)"]
    SEND --> ENUM["EnumWindows() locate WorkerW sibling"]
    ENUM --> CHECK{"WorkerW located & icons visible?"}
    
    CHECK -->|Yes (Icons Visible)| ATTACH_W["WorkerW Primary Mode\nSetParent(PlayerHWND, WorkerW)"]
    CHECK -->|No (Win 11 24H2 Hidden Icons)| ATTACH_P["Progman Fallback Mode\nAttach beneath SHELLDLL_DefView"]
    
    ATTACH_W --> STYLE["SetWindowLongPtrW(WS_CHILD | WS_CLIPSIBLINGS)"]
    ATTACH_P --> STYLE
    STYLE --> POSITION["SetWindowPos(HWND, HWND_BOTTOM, MonitorRect)"]
    POSITION --> MONITOR["Listen for TaskbarCreated message"]
    
    MONITOR -->|Explorer Crashes| RECOVER["Detect handle invalidation\nWait for TaskbarCreated\nRe-attach within 2.5s"]
    MONITOR -->|Normal Engine Exit| RESTORE["Detach HWND\nSystemParametersInfoW(SPI_SETDESKWALLPAPER)\nRestore default static wallpaper"]

    classDef proc fill:#003566,stroke:#ffc300,stroke-width:2px,color:#ffd60a;
    class START,SEND,ENUM,ATTACH_W,ATTACH_P,STYLE,POSITION,MONITOR,RECOVER,RESTORE proc;
```

### Dual-Mode Attachment Logic (`DEC-PLT-001`)
1. **Primary WorkerW Mode:** Broadcasts `0x052C` to `Progman`. Enumerates top-level windows to locate the `WorkerW` spawned directly beneath `SHELLDLL_DefView`. Reparents the player `HWND` into `WorkerW`. Desktop icons remain 100% clickable and interactive.
2. **Fallback Progman Mode (Windows 11 24H2):** When desktop icons are hidden ("Show desktop icons" unchecked), Windows 11 24H2 destroys the secondary `WorkerW`. The engine detects this condition and reparents the player `HWND` directly as a child of `Progman` beneath `SHELLDLL_DefView`.
3. **Explorer Crash Resilience (`KW-F-003`):** Registers `TaskbarCreated` via `RegisterWindowMessageW`. When Explorer restarts, the daemon detects handle invalidation, waits for `TaskbarCreated`, re-spawns `WorkerW`, and re-attaches players within ≤2.5 seconds.

