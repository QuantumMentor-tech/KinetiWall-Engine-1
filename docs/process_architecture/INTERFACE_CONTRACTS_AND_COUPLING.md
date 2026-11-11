# KinetiWall — Interface Contract Catalog & Coupling Prevention Register
**Document ID:** `KW-PRC-007` & `KW-PRC-008`  
**Deliverables:** Deliverable G (Interface Contracts) & Deliverable H (Coupling & Decisions)  
**Phase:** 2 — System Architecture & Technical Decisions  
**Step:** 8 of 90 — Multi-Process Boundaries & Component Responsibilities  
**Status:** COMPLETE (Authoritative Interface & Coupling Governance)  
**Authority:** Principal Software Architect & IPC Systems Lead  
**Date:** 2026-11-10  

---

## 1. Deliverable G: Cross-Component Interface Contract Catalog

The catalog below defines the 12 formal interface contracts governing all communication across the core controller, gallery UI, player subprocesses, and the Windows shell:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                 12 CROSS-COMPONENT INTERFACE CONTRACTS                      │
├─────────┬─────────────────────────────┬──────────────┬──────────────────────┤
│ Cont ID │ Contract Name               │ Producer     │ Consumer             │
├─────────┼─────────────────────────────┼──────────────┼──────────────────────┤
│ CTR-01  │ QueryApplicationState       │ Gallery UI   │ Core Daemon          │
│ CTR-02  │ ApplyWallpaperToMonitor     │ Gallery UI   │ Core Daemon          │
│ CTR-03  │ UpdateCustomProperty        │ Gallery UI   │ Core -> Player Web   │
│ CTR-04  │ SetGlobalPauseState         │ UI / Policy  │ Core -> All Players  │
│ CTR-05  │ StopAndReleaseResources     │ Core Daemon  │ Player Subprocesses  │
│ CTR-06  │ ReportPlayerReadyHandshake  │ Player Proc  │ Core Daemon          │
│ CTR-07  │ ReportPlaybackStateAck      │ Player Proc  │ Core (StateCoord)    │
│ CTR-08  │ ReportPlaybackFailure       │ Player Proc  │ Core (ProcessOrch)   │
│ CTR-09  │ NotifyTopologyDelta         │ Core (DTM)   │ Gallery UI & Players │
│ CTR-10  │ StreamAudioSpectrumArray    │ Core (Audio) │ Web Wallpaper Player │
│ CTR-11  │ ImportArchivePackage        │ Gallery UI   │ Core (LibraryRepo)   │
│ CTR-12  │ RequestSafePlayerTeardown   │ Core Daemon  │ Player Subprocesses  │
└─────────┴─────────────────────────────┴──────────────┴──────────────────────┘
```

---

### Exhaustive Interface Contract Specifications

#### `CTR-01`: `QueryApplicationState`
- **Producer:** `kinetiwall-ui.exe` | **Consumer:** `kinetiwall-core.exe`
- **Purpose:** Fetches authoritative snapshot of active displays, assigned wallpapers, volume levels, and global pause flags.
- **Inputs:** `QueryStateRequest { client_version: String }`
- **Outputs:** `ApplicationStateSnapshot { displays: Vec<DisplayInfo>, assignments: Map<DisplayId, WallpaperProfile>, global_pause: bool }`
- **Preconditions:** Named Pipe connection established; interactive user SID authenticated.
- **Postconditions:** UI renders active dashboard; zero daemon state mutated.
- **Failure Modes / Timeout:** Pipe timeout (3.0s); returns `IPC_ERROR_TIMEOUT`.
- **Security:** Read-only access; non-destructive.
- **Related Acceptance Criteria:** `KW-AC-005`, `KW-AC-014`.

#### `CTR-02`: `ApplyWallpaperToMonitor`
- **Producer:** `kinetiwall-ui.exe` | **Consumer:** `kinetiwall-core.exe`
- **Purpose:** Assigns a validated wallpaper profile from local library to a specific physical display.
- **Inputs:** `ApplyWallpaperCommand { display_id: u32, wallpaper_id: String, mode: PresentationMode }`
- **Outputs:** `CommandResponse { success: bool, message: Option<String> }`
- **Preconditions:** Target `wallpaper_id` exists in `%LOCALAPPDATA%\KinetiWall\wallpapers\`; `display_id` is active.
- **Postconditions:** Core updates `WallpaperAssignmentManager`, saves `config.json`, terminates existing player on display, spawns new player.
- **Failure Modes / Timeout:** If media file unreadable, returns `ERROR_FILE_NOT_FOUND` without spawning process.
- **Security:** Verified local path strictly within user AppData; rejects arbitrary paths.
- **Related Acceptance Criteria:** `KW-AC-009`, `KW-AC-012`, `KW-AC-014`.

#### `CTR-03`: `UpdateCustomProperty`
- **Producer:** `kinetiwall-ui.exe` | **Consumer:** `kinetiwall-core.exe` $\rightarrow$ `kinetiwall-player-web.exe`
- **Purpose:** Updates dynamic wallpaper properties (colors, sliders, toggles) defined in `project.json` without reloading page.
- **Inputs:** `UpdatePropertyCommand { display_id: u32, property_key: String, property_value: JsonValue }`
- **Outputs:** `PropertyUpdatedAck { display_id: u32, key: String }`
- **Preconditions:** Web wallpaper active on target display; property exists in manifest schema.
- **Postconditions:** Core forwards command to web player; web player executes `window.chrome.webview.postMessage`; canvas updates in <50ms.
- **Failure Modes / Timeout:** If property key invalid, returns `ERROR_PROPERTY_NOT_FOUND`.
- **Security:** Value checked against schema type (prevents string injection into numeric uniforms).
- **Related Acceptance Criteria:** `KW-AC-013`.

#### `CTR-04`: `SetGlobalPauseState`
- **Producer:** `ResourcePolicyEngine` / Tray Menu / UI | **Consumer:** Core $\rightarrow$ All Active Players
- **Purpose:** Halts or resumes rendering presentation across all display viewports.
- **Inputs:** `SetPauseCommand { pause: bool, immediate_freeze: bool }`
- **Outputs:** `PauseCommandAck { paused_displays: Vec<u32> }`
- **Preconditions:** Active players running.
- **Postconditions:** Video players halt D3D11 swapchain present (0.00% GPU); web players set `put_IsVisible(FALSE)`.
- **Failure Modes / Timeout:** Timeout: 50ms. If player unresponsive, core logs warning and drops priority.
- **Security:** Enforced only by core daemon authority.
- **Related Acceptance Criteria:** `KW-AC-010`, `KW-AC-016`.

#### `CTR-05`: `StopAndReleaseResources`
- **Producer:** Core Daemon | **Consumer:** Player Subprocesses
- **Purpose:** Completely releases GPU VRAM, video decoders, and WebView2 instances during extended idle or sleep.
- **Inputs:** `StopRequest { display_id: u32 }`
- **Outputs:** `StopAck { freed_vram_mb: u32 }`
- **Preconditions:** Player in paused state for >15 minutes or system entering sleep.
- **Postconditions:** Player destroys swapchain and unloads media; blits static image fallback to desktop.
- **Failure Modes / Timeout:** If player hangs, core terminates child process via Job Object.
- **Security:** Clean memory de-allocation; zero handle leaks.
- **Related Acceptance Criteria:** `KW-AC-018`, `KW-MET-006`.

#### `CTR-06`: `ReportPlayerReadyHandshake`
- **Producer:** Player Subprocess (`video/web`) | **Consumer:** Core Controller
- **Purpose:** Signals that child rendering window is created and ready for desktop reparenting.
- **Inputs:** `PlayerReadyMessage { pid: u32, display_id: u32, surface_hwnd: u64 }`
- **Outputs:** `AttachmentCompleteAck { parent_hwnd: u64, rect: Rect }`
- **Preconditions:** Player created child `HWND`; core listening on Named Pipe.
- **Postconditions:** Core verifies `GetWindowThreadProcessId(surface_hwnd) == pid`, executes `SetParent`, and sets window position.
- **Failure Modes / Timeout:** If HWND does not belong to child PID, core rejects connection and kills rogue process.
- **Security:** Eliminates handle-spoofing and unauthorized window reparenting.
- **Related Acceptance Criteria:** `KW-AC-001`, `KW-AC-009`.

#### `CTR-07`: `ReportPlaybackStateAck`
- **Producer:** Player Subprocess | **Consumer:** Core (`StateCoordinator`)
- **Purpose:** Confirms actual observed physical playback state ($S_{\text{observed}}$) back to controller.
- **Inputs:** `PlaybackStateAck { display_id: u32, state: ObservedState, frame_time_ms: f32 }`
- **Outputs:** None (One-way telemetry ACK).
- **Preconditions:** Player completes state transition.
- **Postconditions:** `StateCoordinator` reconciles desired vs observed state; updates public status snapshot.
- **Failure Modes / Timeout:** If ACK missing >2.0s, core marks display `Recovering`.
- **Security:** High-frequency lightweight telemetry packet (<128 bytes).
- **Related Acceptance Criteria:** `KW-MET-009`, `KW-MET-010`.

#### `CTR-08`: `ReportPlaybackFailure`
- **Producer:** Player Subprocess | **Consumer:** Core (`ProcessOrchestrator`)
- **Purpose:** Reports unrecoverable media decoding, WebGL context loss, or driver TDR error.
- **Inputs:** `PlaybackFailureEvent { display_id: u32, error_code: u32, error_message: String }`
- **Outputs:** `FailureActionDirective { action: Retry | Abort }`
- **Preconditions:** Decode error in libmpv or WebGL crash in WebView2.
- **Postconditions:** Core increments crash counter, triggers exponential backoff restart, or switches to static fallback.
- **Failure Modes / Timeout:** Handled immediately by supervisor thread.
- **Security:** Error string sanitized to 1 KB plain text (prevents log injection).
- **Related Acceptance Criteria:** `KW-AC-008`, `KW-NF-009`.

#### `CTR-09`: `NotifyTopologyDelta`
- **Producer:** Core (`DisplayTopologyManager`) | **Consumer:** Gallery UI & Players
- **Purpose:** Pushes display connect, disconnect, or DPI scale factor changes to active components.
- **Inputs:** `TopologyDeltaEvent { connected: Vec<DisplayInfo>, disconnected: Vec<u32>, dpi_scales: Map<u32, f32> }`
- **Outputs:** None (Broadcast notification).
- **Preconditions:** Win32 `WM_DISPLAYCHANGE` or `WM_DPICHANGED` caught by core daemon.
- **Postconditions:** Players re-size swapchain buffers; UI re-renders monitor layout grid.
- **Failure Modes / Timeout:** Non-blocking broadcast.
- **Security:** Core-originated signed broadcast.
- **Related Acceptance Criteria:** `KW-AC-015`.

#### `CTR-10`: `StreamAudioSpectrumArray`
- **Producer:** Core (`AudioCoordinator`) | **Consumer:** Web Wallpaper Player
- **Purpose:** Streams 64-band frequency arrays at 60 FPS to audio-reactive web wallpapers.
- **Inputs:** None (Direct shared memory ring buffer read).
- **Outputs:** `FrequencySpectrum { bands: [f32; 64], timestamp: u64 }`
- **Preconditions:** Audio reactivity enabled; system audio playing; WASAPI loopback active.
- **Postconditions:** Web canvas animates audio bars in synchronization (<20ms latency).
- **Failure Modes / Timeout:** If audio pauses, buffer flushes to 0.0 magnitudes.
- **Security:** Strictly volatile memory; 0 bytes written to disk; 0 network packets (`KW-NG-005`).
- **Related Acceptance Criteria:** `KW-AC-021`, `KW-NF-015`.

#### `CTR-11`: `ImportArchivePackage`
- **Producer:** Gallery UI | **Consumer:** Core (`LibraryRepository`)
- **Purpose:** Validates and unpacks imported `.kwall` / `.zip` wallpaper archives into local storage.
- **Inputs:** `ImportArchiveRequest { source_path: PathBuf }`
- **Outputs:** `ImportArchiveResponse { success: bool, new_wallpaper_id: Option<String>, error: Option<String> }`
- **Preconditions:** User drags file into UI; file exists on disk.
- **Postconditions:** Core canonicalizes archive paths, verifies Zip Slip protection (`SEC-03`), parses `project.json`, generates thumbnails.
- **Failure Modes / Timeout:** Traversal path (`../`) detected $\rightarrow$ extraction aborted, file deleted.
- **Security:** Enforces strict filesystem sandboxing.
- **Related Acceptance Criteria:** `KW-AC-019`, `KW-NF-014`.

#### `CTR-12`: `RequestSafePlayerTeardown`
- **Producer:** Core Daemon | **Consumer:** Player Subprocesses
- **Purpose:** Instructs child player to perform clean shutdown and window detachment.
- **Inputs:** `TeardownDirective { graceful_timeout_ms: u32 }`
- **Outputs:** `TeardownCompleteAck`
- **Preconditions:** Engine shutdown, wallpaper change, or display disconnect.
- **Postconditions:** Player destroys D3D11 swapchain, releases window handles, and terminates process.
- **Failure Modes / Timeout:** If child does not exit in 300ms, core terminates process via `TerminateProcess` / Job Object.
- **Security:** Eliminates hanging processes and handle leaks.
- **Related Acceptance Criteria:** `KW-AC-004`, `KW-AC-007`.

---

## 2. Deliverable H: Coupling Prevention Register

To guarantee long-term maintainability, the proposed process topology was audited against 13 critical architectural coupling hazards:

| Hazard ID | Architectural Anti-Pattern | Potential Consequence | Enforced KinetiWall Architecture Rule |
|:---|:---|:---|:---|
| **`CPL-01`** | **Multiple State Authorities** | UI and Core disagree on whether wallpaper is playing; race conditions. | `StateCoordinator` inside `kinetiwall-core.exe` is the sole source of truth. UI is strictly a read-only mirror. |
| **`CPL-02`** | **UI Directly Touching Win32 HWNDs** | Broken desktop attachment when UI closes; invalid handle panics. | UI has zero desktop window access. Only `DesktopHostManager` in `core` manages `WorkerW` / `Progman` handles. |
| **`CPL-03`** | **Players Mutating Persistent Config** | Concurrent write race conditions; corrupted `config.json`. | Players have zero write access to configuration. Only `ConfigurationRepository` in `core` writes to disk. |
| **`CPL-04`** | **Wallpaper JS Invoking Native APIs** | Malicious web script spawns `cmd.exe` or deletes user files. | Zero native execution bridge exposed to DOM. Communication limited strictly to sanitized JSON properties via `postMessage`. |
| **`CPL-05`** | **Circular Startup Dependencies** | Core waits for UI while UI waits for Core; deadlock on startup. | Core starts 100% independently as a background daemon. UI is an optional on-demand client launched later. |
| **`CPL-06`** | **Excessive High-Frequency IPC** | IPC pipe saturated by 60 FPS video frame telemetry; CPU spikes. | Video decoding and presentation loops remain internal to the player. IPC transmits only state transitions and errors. |
| **`CPL-07`** | **Duplicated Resource Management** | UI and Core both attempting to kill or restart the same player. | Only `ProcessOrchestrator` in `core` possesses process handles and authority to restart or kill players. |
| **`CPL-08`** | **Independent Player Restart Racing** | Multiple components spawning duplicate players for the same display. | `ProcessOrchestrator` maintains an exclusive 1:1 map between `DisplayId` and active player PID. |
| **`CPL-09`** | **Unsynchronized Multi-Writer Storage** | Corrupted database or manifest files during simultaneous imports. | All library modifications and imports are routed serially through `LibraryRepository` in `core`. |
| **`CPL-10`** | **Stale Window Handles Retained** | Operations dispatched to destroyed Explorer HWNDs crash the engine. | `DesktopHostManager` invalidates all parent HWNDs upon Explorer crash and refuses operations until re-hooked. |
| **`CPL-11`** | **Core Performing Heavy Rendering** | Controller event loop hangs during video decode; tray freezes. | Core daemon never touches video decoding or WebGL. Heavy rendering is 100% offloaded to player subprocesses. |
| **`CPL-12`** | **Unbounded Crash Recovery Loops** | A corrupt video file causes player to restart in an infinite CPU loop. | Bounded backoff: maximum 3 restarts in 60s; permanent static wallpaper fallback if threshold breached (`KW-F-007`). |
| **`CPL-13`** | **Optional Features Adding Processes** | AI or RGB features creating permanent background processes. | All optional features remain strictly scoped to V1.x/V2.0+ candidates and execute on-demand without baseline bloat. |

---

## 3. Architecture Decision Updates & Linkage

All process boundaries defined in this specification directly align with the Architecture Decision Records established in Step 7:
- **`ADR-0002` (User-Session Daemon):** Implemented in `PROC-01` (`kinetiwall-core.exe`).
- **`ADR-0003` (Multi-Process Isolation):** Implemented across `PROC-01`, `PROC-03`, and `PROC-04`.
- **`ADR-0004` (Ephemeral UI 0 MB Closed):** Implemented in `PROC-02` (`kinetiwall-ui.exe`).
- **`ADR-0006` (Embedded libmpv D3D11VA):** Implemented in `PROC-03` (`kinetiwall-player-video.exe`).
- **`ADR-0007` & `ADR-0013` (WebView2 Sandbox):** Implemented in `PROC-04` (`kinetiwall-player-web.exe`).
- **`ADR-0008` (Named Pipes with DACL):** Implemented in `IPC-01` through `IPC-05`.
- **`ADR-0010` (Windows Job Objects):** Implemented in `ProcessOrchestrator` (`KILL_ON_JOB_CLOSE`).
- **`ADR-0014` (3D Scoped to WebGL):** Scoped to `PROC-04`; `PROC-05` (standalone .exe reparenting) deferred to V2.0+ pending `VG-009`.

