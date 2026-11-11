# KinetiWall — Lifecycle & Failure Responsibility Specification
**Document ID:** `KW-PRC-006`  
**Deliverable:** Deliverable F — Lifecycle and Failure Responsibility Specification  
**Phase:** 2 — System Architecture & Technical Decisions  
**Step:** 8 of 90 — Multi-Process Boundaries & Component Responsibilities  
**Status:** COMPLETE (Authoritative Lifecycle & Recovery Blueprint)  
**Authority:** Principal Software Architect & Reliability Engineering Lead  
**Date:** 2026-11-10  

---

## 1. Lifecycle Responsibility Allocation Matrix

To prevent competing components from mishandling process transitions, KinetiWall explicitly assigns ownership for every stage of application lifecycle and failure recovery:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                 LIFECYCLE RESPONSIBILITY ALLOCATION                         │
├─────────────────────┬─────────────────────┬─────────────────────────────────┤
│ Lifecycle Stage     │ Responsible Module  │ Delegated Operational Action    │
├─────────────────────┼─────────────────────┼─────────────────────────────────┤
│ Core Boot & Mutex   │ AppLifecycleManager │ Single instance lock, tray init │
│ Config Bootstrap    │ ConfigurationRepo   │ Atomic config.json load/restore │
│ Shell Discovery     │ DesktopHostManager  │ 0x052C broadcast, WorkerW probe │
│ Display Discovery   │ DisplayTopologyMgr  │ EnumDisplayMonitors rect map    │
│ Player Orchestration│ ProcessOrchestrator │ Job Object spawn, backoff retry │
│ State Reconciliation│ StateCoordinator    │ Desired vs Observed state audit │
│ Explorer Recovery   │ DesktopHostManager  │ TaskbarCreated re-attachment    │
│ Clean Teardown      │ AppLifecycleManager │ Job termination, static restore │
└─────────────────────┴─────────────────────┴─────────────────────────────────┘
```

---

## 2. Exhaustive Lifecycle Workflow Specifications

### 2.1 Application Startup Coordination
- **Step 1 (Single-Instance):** `AppLifecycleManager` creates named mutex `Global\KinetiWall_Core_SingleInstance`. If already held, connects to Named Pipe, dispatches `OpenGalleryRequest`, and exits immediately.
- **Step 2 (Configuration Load):** `ConfigurationRepository` parses `config.json`. If corrupted, loads `config.json.bak` and logs warning.
- **Step 3 (IPC Endpoint):** `IpcServer` starts Named Pipe `\\.\pipe\kinetiwall-ipc-{SessionId}` with logon SID DACL.
- **Step 4 (Shell Probing):** `DesktopHostManager` sends `0x052C` to `Progman`; checks icon visibility; establishes `WorkerW` or `Progman` attachment mode (`DEC-PLT-001`).
- **Step 5 (Display Discovery):** `DisplayTopologyManager` enumerates monitors via `EnumDisplayMonitors`, constructs virtual coordinate rectangle map.
- **Step 6 (Policy Hooks):** `ResourcePolicyEngine` registers `SetWinEventHook` for `EVENT_SYSTEM_FOREGROUND`.
- **Step 7 (Player Spawning):** `ProcessOrchestrator` inspects assigned wallpapers per display, spawns required child players inside Windows Job Object (`JOB_OBJECT_LIMIT_KILL_ON_JOB_CLOSE`).
- **Step 8 (Attachment Handshake):** Players initialize D3D11/WebView2 and return `PlayerReady(HWND)`; `DesktopHostManager` reparents window beneath shell host.
- **Prerequisite Boundary:** If Step 1, 3, or 4 fails, startup halts with diagnostic error; failure of an individual display player (Step 7) does **not** halt the core daemon.

---

### 2.2 Gallery UI Launch & Ephemeral Teardown
- **Launch Request:** User clicks tray icon or shortcut. `AppLifecycleManager` calls `CreateProcessW` to spawn `kinetiwall-ui.exe`.
- **IPC Handshake:** UI connects to Named Pipe, requests state. Core daemon returns display geometry, active profiles, and library index.
- **Mid-Flight Termination Handling:** If the user closes the Gallery window while an archive import or setting change is in progress, the UI sends `GalleryClosing` notification and terminates via `std::process::exit(0)`. The `IpcServer` flushes the pipe without panicking; the core completes the pending atomic file write safely.
- **Zero Resident Footprint:** UI process vanishes from Task Manager within ≤200ms; releases 100% of its 70–120 MB RAM (`KW-NG-001`). Active wallpapers continue playing uninterrupted.

---

### 2.3 Player Crash Recovery & Bounded Auto-Restart
- **Detection:** `ProcessOrchestrator` maintains an async wait thread pool on child process handles (`RegisterWaitForSingleObject`).
- **Crash Catch:** When a child process exits with non-zero exit code (e.g. GPU TDR or codec crash), the wait callback fires immediately.
- **State Transition:** `StateCoordinator` updates display state: `Playing` $\rightarrow$ `Recovering`.
- **Restart Algorithm:**
  1. If `crash_count == 0`: Wait 100ms, spawn replacement player.
  2. If `crash_count == 1`: Wait 500ms, spawn replacement player.
  3. If `crash_count == 2`: Wait 1,500ms, spawn replacement player.
  4. If `crash_count >= 3` within 60s: Halt restart loop! Transition display state to `Failed`.
- **Fallback Action:** Revert monitor to Windows static wallpaper; log diagnostic incident; notify user via tray toast.

---

### 2.4 Controller Crash or Exit (Zero-Orphan Guarantee)
- **Problem:** What prevents child players from lingering if the controller is terminated forcefully via Task Manager?
- **Enforcement:** All child player processes (`PROC-03`, `PROC-04`) are assigned to a Windows Job Object configured with `JOB_OBJECT_LIMIT_KILL_ON_JOB_CLOSE` immediately upon creation.
- **Kernel Mechanism:** The Windows NT kernel tracks the job handle. When `kinetiwall-core.exe` terminates, the operating system kernel closes the job object handle, which triggers **unconditional termination of all child processes assigned to the job**. Exactly 0 orphan processes remain.
- **Desktop Restoration:** On graceful shutdown, `DesktopHostManager` restores desktop wallpaper via `SystemParametersInfoW(SPI_SETDESKWALLPAPER)`. On hard crash, desktop icons remain over the native static background.

---

### 2.5 Windows Explorer Shell Crash & Re-Attachment
- **Detection:** Windows broadcasts `TaskbarCreated` registered window message when a restarted `explorer.exe` initializes the taskbar.
- **Invalidation:** The termination of the previous `explorer.exe` invalidates existing `WorkerW` and `Progman` HWNDs. `DesktopHostManager` marks desktop handles stale and dispatches `PauseRendering` to players.
- **Recovery Procedure:**
  1. Core receives `TaskbarCreated`.
  2. Re-broadcasts `0x052C` to the new `Progman`.
  3. Locates newly spawned `WorkerW`.
  4. Re-parents active player `HWND`s beneath new `WorkerW` via `SetParent`.
  5. Positions windows via `SetWindowPos(HWND, HWND_BOTTOM, rect)`.
  6. Dispatches `ResumeRendering` to players.
- **Execution Budget:** Entire recovery completes within ≤2.5 seconds (`KW-F-003`).

---

### 2.6 Graphics Driver Reset (TDR) & Display Disconnect
- **GPU TDR Event:** When Windows resets the display driver (`DXGI_ERROR_DEVICE_RESET` / `DEVICE_REMOVED`), the child player catches the error on `Present()`, releases all D3D11 swapchains and texture views, recreates the D3D11 device on the new adapter, and resumes rendering without crashing.
- **Display Disconnect (Hotplug):** When a monitor is unplugged, `DisplayTopologyManager` catches `WM_DISPLAYCHANGE`. `ProcessOrchestrator` immediately terminates the player assigned to the removed display, freeing its VRAM and RAM. Surviving display players re-anchor viewports dynamically.

