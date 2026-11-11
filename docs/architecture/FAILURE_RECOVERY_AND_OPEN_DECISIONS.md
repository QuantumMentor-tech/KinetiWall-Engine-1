# KinetiWall — Failure Recovery Architecture & Architecture Risk Register
**Document ID:** `KW-ARC-007` & `KW-ARC-008`  
**Deliverables:** Deliverable G (Failure Recovery) & Deliverable H (Architecture Risk Register)  
**Phase:** 2 — System Architecture & Technical Decisions  
**Step:** 7 of 90 — System Architecture & Architecture Decision Records  
**Status:** COMPLETE (Auditable Failure & Risk Governance)  
**Authority:** Principal Software Architect & QA Systems Architect  
**Date:** 2026-11-10  

---

## 1. Deliverable G: 16 Critical Failure Recovery Scenarios

The matrix below models the system's deterministic recovery behavior across all 16 critical failure scenarios specified in Section 16:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                 16 CRITICAL ARCHITECTURAL FAILURE SCENARIOS                 │
├─────────┬─────────────────────────────┬──────────────┬──────────────────────┤
│ Case ID │ Failure Event               │ Responsible  │ Deterministic Action │
├─────────┼─────────────────────────────┼──────────────┼──────────────────────┤
│ FAIL-01 │ Core Startup Failure        │ AppLifecycle │ Log error, exit cleanly│
│ FAIL-02 │ Gallery UI Mid-IPC Crash    │ IpcServer    │ Flush pipe, keep play│
│ FAIL-03 │ Video Player Process Crash  │ ProcessOrch  │ Auto-restart (max 3) │
│ FAIL-04 │ Web Player Unresponsive/Leak│ JobObject    │ 500 MB quota kill    │
│ FAIL-05 │ IPC Connection Interrupted  │ IpcServer    │ Re-listen on pipe    │
│ FAIL-06 │ Explorer Shell Restarts     │ DesktopHost  │ TaskbarCreated rehook│
│ FAIL-07 │ Desktop Attachment Fails    │ DesktopHost  │ Progman fallback     │
│ FAIL-08 │ Monitor Disappears (Hotplug)│ TopologyMgr  │ Destroy child player │
│ FAIL-09 │ Graphics Device Reset (TDR) │ Players      │ Recreate D3D11 device│
│ FAIL-10 │ WebView2 Runtime Missing    │ WebPlayer    │ Disable web, keep vid│
│ FAIL-11 │ Corrupt Media Decode Error  │ VideoPlayer  │ Static preview blit  │
│ FAIL-12 │ Malformed Package Manifest  │ LibraryRepo  │ Quarantine bad item  │
│ FAIL-13 │ Audio Endpoint Disconnected │ AudioEngine  │ Auto-rebind <500ms   │
│ FAIL-14 │ Unexpected Child Process Exit│ ProcessOrch │ Backoff restart      │
│ FAIL-15 │ Normal Exit with Players Run│ AppLifecycle │ Destroy HWND, kill job│
│ FAIL-16 │ Power Loss Mid-Config Write │ ConfigRepo   │ Restore from .bak    │
└─────────┴─────────────────────────────┴──────────────┴──────────────────────┘
```

---

### Detailed Failure Scenario Specifications

#### `FAIL-01`: Core Fails During Startup
- **Detection:** Unhandled panic in `AppLifecycleManager` or mutex acquisition failure.
- **Responsible Component:** `AppLifecycleManager`
- **State Transition:** `Bootstrapping` $\rightarrow$ `Failed` $\rightarrow$ Process Exit.
- **Resource Cleanup:** Releases Named Pipe handle, restores desktop static wallpaper via global panic hook.
- **User-Visible Behavior:** Desktop remains default static wallpaper; native error message box displayed if interactive.
- **Recovery Policy:** Write error to `%LOCALAPPDATA%\KinetiWall\logs\crash.log`; exit with non-zero exit code.

#### `FAIL-02`: Gallery UI Fails During an IPC Request
- **Detection:** Pipe read returns `ERROR_BROKEN_PIPE` or unexpected client EOF.
- **Responsible Component:** `IpcServer`
- **State Transition:** None (Wallpaper playback state is unaffected).
- **Resource Cleanup:** Disconnects and flushes named pipe handle via `DisconnectNamedPipe`; re-enters listening state.
- **User-Visible Behavior:** Gallery window closes abruptly; wallpapers continue playing smoothly on all displays.
- **Recovery Policy:** User simply re-opens the gallery from tray icon; core daemon state remains 100% intact.

#### `FAIL-03`: Video Player Crashes Mid-Playback
- **Detection:** Async thread wait detects signaled child process handle with non-zero exit code.
- **Responsible Component:** `ProcessOrchestrator`
- **State Transition:** `Playing` $\rightarrow$ `Loading` (Attempt 1–3) $\rightarrow$ `Failed` (if >3).
- **Resource Cleanup:** Closes invalid process and thread handles; frees associated pipe session.
- **User-Visible Behavior:** Frame freezes briefly; video restarts within ≤1.5s. If repeated 3 times, monitor reverts to static wallpaper.
- **Recovery Policy:** Exponential backoff restart (100ms, 500ms, 1500ms). Quarantines failing profile after 3 attempts.

#### `FAIL-04`: Web Player Becomes Unresponsive or Leaks Memory
- **Detection:** Windows Job Object triggers `JOB_OBJECT_MSG_PROCESS_MEMORY_LIMIT` (breaching 500 MB quota) or IPC heartbeat timeout (>5s).
- **Responsible Component:** Windows Job Object / `ProcessOrchestrator`
- **State Transition:** `Playing` $\rightarrow$ `Failed`.
- **Resource Cleanup:** Kernel terminates runaway web player process; Job Object releases all committed memory.
- **User-Visible Behavior:** Web animation halts; desktop falls back to static wallpaper; toast notification displayed.
- **Recovery Policy:** Player is not automatically restarted to prevent repeating the memory exhaustion loop.

#### `FAIL-05`: IPC Connection Is Interrupted
- **Detection:** Heartbeat ping timeout or IO completion failure on Named Pipe.
- **Responsible Component:** `IpcServer`
- **State Transition:** IPC channel resets; engine state remains preserved.
- **Resource Cleanup:** Closes broken pipe instance; calls `ConnectNamedPipe` for new incoming connection.
- **User-Visible Behavior:** None.
- **Recovery Policy:** Re-establish pipe listener within 50ms; client reconnects automatically.

#### `FAIL-06`: Windows Explorer Restarts
- **Detection:** Receipt of registered window message `TaskbarCreated` or `WorkerW` HWND invalidation.
- **Responsible Component:** `DesktopHostManager`
- **State Transition:** `Playing` $\rightarrow$ `Detached` $\rightarrow$ `Recovering` $\rightarrow$ `Playing`.
- **Resource Cleanup:** Invalidates stale HWND handles; signals players to halt presentation temporarily.
- **User-Visible Behavior:** Desktop icons flicker as Explorer boots; wallpaper re-appears behind icons within ≤2.5s.
- **Recovery Policy:** Broadcast `0x052C` to new `Progman`, locate new `WorkerW`, re-parent player windows.

#### `FAIL-07`: Desktop Attachment Fails
- **Detection:** `WorkerW` not located after 3 probe iterations of message `0x052C`.
- **Responsible Component:** `DesktopHostManager`
- **State Transition:** `Initializing` $\rightarrow$ `Progman Fallback` (or `Failed`).
- **Resource Cleanup:** Destroys orphaned child window.
- **User-Visible Behavior:** Engine engages Fallback Mode beneath `Progman` child hierarchy (`DEC-PLT-001`).
- **Recovery Policy:** If both WorkerW and Progman fallback fail, logs error and leaves desktop static wallpaper intact.

#### `FAIL-08`: A Physical Monitor Disappears (Hotplug Removal)
- **Detection:** `WM_DISPLAYCHANGE` window message; `EnumDisplayMonitors` returns reduced monitor count.
- **Responsible Component:** `DisplayTopologyManager` & `ProcessOrchestrator`
- **State Transition:** `Playing` $\rightarrow$ `Terminated` (for disconnected display).
- **Resource Cleanup:** Immediately destroys child player process bound to the removed display; frees VRAM.
- **User-Visible Behavior:** Surviving displays continue playing uninterrupted; disconnected display player cleanly removed.
- **Recovery Policy:** Update in-memory display array; if display is reconnected later, re-spawn assigned wallpaper.

#### `FAIL-09`: Graphics Device Reset (TDR Event)
- **Detection:** DXGI present call returns `DXGI_ERROR_DEVICE_RESET` or `DXGI_ERROR_DEVICE_REMOVED`.
- **Responsible Component:** Wallpaper Player Subprocesses (`libmpv` / WebView2)
- **State Transition:** `Playing` $\rightarrow$ `Loading` $\rightarrow$ `Playing`.
- **Resource Cleanup:** Releases all D3D11 swapchains, render target views, and texture allocations.
- **User-Visible Behavior:** Screen flashes black for ~500ms as Windows resets display driver; wallpaper resumes playing.
- **Recovery Policy:** Player recreates D3D11 device and swapchain on new adapter handle; resumes playback loop.

#### `FAIL-10`: WebView2 Runtime Is Unavailable
- **Detection:** `CreateCoreWebView2EnvironmentWithOptions` returns `HRESULT` error (`HRESULT_FROM_WIN32(ERROR_FILE_NOT_FOUND)`).
- **Responsible Component:** `kinetiwall-player-web.exe`
- **State Transition:** `Loading` $\rightarrow$ `Failed`.
- **Resource Cleanup:** Terminates web player subprocess cleanly.
- **User-Visible Behavior:** Video wallpapers continue playing normally; UI disables web wallpaper selection with actionable warning: "WebView2 Runtime Required".
- **Recovery Policy:** Log runtime missing event; offer download link in settings.

#### `FAIL-11`: Corrupt Media File / Codec Decode Failure
- **Detection:** `libmpv` emits `MPV_EVENT_END_FILE` with error code `MPV_END_FILE_REASON_ERROR`.
- **Responsible Component:** `kinetiwall-player-video.exe`
- **State Transition:** `Loading` $\rightarrow$ `Failed`.
- **Resource Cleanup:** Closes video demuxer; halts D3D11 decode engine.
- **User-Visible Behavior:** Desktop displays static preview thumbnail; UI highlights wallpaper with error badge.
- **Recovery Policy:** Do not loop-restart bad media; prompt user to select valid file.

#### `FAIL-12`: Malformed Wallpaper Package Manifest
- **Detection:** `serde_json::from_str` returns syntax or schema validation error during `project.json` parse.
- **Responsible Component:** `LibraryRepository`
- **State Transition:** None (Import rejected).
- **Resource Cleanup:** Deletes temporary extraction files.
- **User-Visible Behavior:** Gallery UI displays toast: "Invalid wallpaper manifest: {error_details}".
- **Recovery Policy:** Quarantine folder; continue scanning rest of library.

#### `FAIL-13`: Audio Endpoint Disconnected (Headphones Unplugged)
- **Detection:** `IMMNotificationClient::OnDefaultDeviceChanged` callback or `WM_DEVICECHANGE`.
- **Responsible Component:** `AudioCaptureEngine`
- **State Transition:** Visualizer pauses capture $\rightarrow$ re-binds $\rightarrow$ resumes.
- **Resource Cleanup:** Releases previous `IAudioClient` and `IAudioCaptureClient` COM interfaces.
- **User-Visible Behavior:** Visualizer bars pause for ~200ms, then resume pulsing to the new default audio output.
- **Recovery Policy:** Re-initialize WASAPI loopback client on new default audio endpoint within ≤500ms.

#### `FAIL-14`: Unexpected Child Process Exit
- **Detection:** Wait on child process handle returns signaled state.
- **Responsible Component:** `ProcessOrchestrator`
- **State Transition:** `Playing` $\rightarrow$ `Recovering`.
- **Resource Cleanup:** Closes process handle; cleans up pipe resources.
- **User-Visible Behavior:** Desktop wallpaper restarts smoothly within ≤1.5s.
- **Recovery Policy:** Bounded exponential backoff auto-restart up to 3 times in 60s.

#### `FAIL-15`: User Exits Application While Players Are Active
- **Detection:** User selects "Exit" from system tray icon menu.
- **Responsible Component:** `AppLifecycleManager`
- **State Transition:** `Playing` $\rightarrow$ `ShuttingDown` $\rightarrow$ Process Exit.
- **Resource Cleanup:** Core instructs players to exit, detaches child `HWND`s, terminates Job Object, restores static wallpaper via `SystemParametersInfoW`.
- **User-Visible Behavior:** Desktop wallpaper smoothly reverts to native Windows background; all KinetiWall processes exit.
- **Recovery Policy:** Complete teardown in ≤300ms.

#### `FAIL-16`: Application Interrupted During Configuration Update
- **Detection:** Power cut or taskkill while saving settings.
- **Responsible Component:** `ConfigurationRepository`
- **State Transition:** Startup recovery checks `config.json` validity.
- **Resource Cleanup:** Deletes orphaned `config.json.tmp`.
- **User-Visible Behavior:** Engine loads successfully using verified `config.json.bak`.
- **Recovery Policy:** Atomic file rename prevents primary file corruption; backup ensures zero data loss.

---

## 2. Deliverable H: Architecture Risk & Open-Decision Register

The register below catalogs the critical architectural decisions requiring experimental validation during Phase 2 prototype spikes:

| Decision ID | Architectural Topic | Current Status | Prototype Validation Gate | Required Evidence for Finalization | Decision Trigger & Reconsideration | Owner Role |
|:---|:---|:---:|:---:|:---|:---|:---:|
| **`DEC-ARC-01`** | **Dual-Mode Desktop Attachment Fallback** | `PROPOSED` | `VG-001` | Screen recordings and Spy++ hierarchy dumps verifying wallpaper renders when desktop icons are hidden on Windows 11 24H2. | If fallback fails, evaluate transparent layered borderless window architecture. | Principal Windows Platform Architect |
| **`DEC-ARC-02`** | **Job Object Child Process Teardown** | `ACCEPTED` | `VG-002` | 50 consecutive hard-kill test runs confirming 0 orphan player processes. | If Job Object fails on specific OS build, implement secondary watchdog service. | Senior Systems Reliability Engineer |
| **`DEC-ARC-03`** | **libmpv D3D11VA Sub-16.6ms Looping** | `ACCEPTED` | `VG-003` | PresentMon frame-time trace proving delta at loop boundary ≤16.6ms with <2.5% 4K CPU usage. | If loop boundary stutters, evaluate dual-player pre-roll buffer swap. | Lead Media Pipeline Engineer |
| **`DEC-ARC-04`** | **Event Hook Fullscreen Occlusion Pause** | `ACCEPTED` | `VG-004` | CapFrameX 99th % FPS delta <0.5% during AAA 3D gaming and ≤50ms pause latency. | If borderless games missed, supplement hook with window rect bounds check. | Senior Performance QA Engineer |
| **`DEC-ARC-05`** | **WebView2 Virtual Host CORS Sandboxing** | `ACCEPTED` | `VG-006` | Penetration test report confirming 100% blocked external network fetch and disk traversal. | If sandbox escapes exist, implement custom local web server with token auth. | Senior Security Research Analyst |
| **`DEC-ARC-06`** | **WASAPI Volatile Audio Loopback Bridge** | `ACCEPTED` | `VG-007` | Process Monitor audit proving 0 bytes written to disk; FFT calculation latency <20ms. | If WASAPI loopback stutters, evaluate Windows Spatial Audio APIs. | Principal Audio Systems Engineer |
| **`DEC-ARC-07`** | **Standalone Offline NSIS Package** | `ACCEPTED` | `VG-008` | Automated installation and immediate execution on air-gapped clean Windows 11 VM. | If prerequisites fail offline, bundle static VC++ runtimes directly. | DevOps & Release Engineering Lead |
| **`DEC-ARC-08`** | **Standalone Game (.exe) Reparenting** | `DEFERRED (V2.0+)` | `VG-009` | Prototype reparenting compiled Godot 4 demo behind desktop icons without input focus loss. | Confirms deferral: 3D delivered via WebGL in V1.0; standalone .exe deferred. | Principal Software Architect |
| **`DEC-ARC-09`** | **LGPL v2.1+ Dynamic Linking Compliance** | `ACCEPTED` | `VG-010` | PE header audit and successful execution of user-replaced custom `libmpv-2.dll`. | If dynamic linking broken, re-architect build pipeline before any distribution. | Open Source Compliance Lead |

