# KinetiWall — State and Resource Ownership Matrix
**Document ID:** `KW-PRC-004`  
**Deliverable:** Deliverable D — State and Resource Ownership Matrix  
**Phase:** 2 — System Architecture & Technical Decisions  
**Step:** 8 of 90 — Multi-Process Boundaries & Component Responsibilities  
**Status:** COMPLETE (Authoritative State & Resource Baseline)  
**Authority:** Principal Software Architect & IPC Systems Lead  
**Date:** 2026-11-10  

---

## 1. Authoritative State Ownership & Invariants

To eliminate state synchronization hazards, KinetiWall enforces a strict **Single Source of Truth (SSOT)** model. No two components may independently maintain or mutate authoritative application state.

### The Desired State vs Observed State Invariant
A foundational flaw in desktop multimedia applications is assuming that requesting a state transition guarantees the transition occurred. KinetiWall explicitly separates:
- **Desired State ($S_{\text{desired}}$):** The target state requested by the user, power policy, or occlusion engine (e.g., "PAUSE Display 1").
- **Observed State ($S_{\text{observed}}$):** The verified physical state confirmed by the player's Direct3D swapchain or WebView2 host via IPC acknowledgment (e.g., "Display 1 Swapchain Frozen").
- **Reconciliation Rule:** If $S_{\text{desired}} \neq S_{\text{observed}}$ for more than 2,000ms, the `StateCoordinator` marks the player `Recovering` and triggers a diagnostic investigation.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                   STATE RECONCILIATION PIPELINE                             │
├─────────────────────────────────────────────────────────────────────────────┤
│ User / Policy Event ──> S_desired (Core) ──> IPC Command ──> Player Subproc │
│                                                                    │        │
│ S_reconciled <── StateCoordinator <── S_observed ACK <── D3D11 Halt         │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Exhaustive Authoritative State Matrix

| State Category | Concrete State Items | Authoritative Owner | Allowed Readers | Allowed Writers | Persistence Rule | Conflict Resolution Strategy |
|:---|:---|:---|:---|:---|:---|:---|
| **Application State** | Engine run-mode (`Bootstrapping`, `Running`, `Exiting`), Global pause override, Mutex lock. | `AppLifecycleManager` (`core`) | All modules, Gallery UI | `AppLifecycleManager` | Transient (In-Memory). | Lifecycle commands take precedence over individual player state. |
| **Wallpaper Assignments** | `DisplayId -> WallpaperId`, Presentation mode (`Independent`, `Span`, `Clone`). | `WallpaperAssignmentManager` (`core`) | Gallery UI, Process Orchestrator | `WallpaperAssignmentManager` via UI IPC | Persistent in `config.json` with `.bak` recovery. | User assignment commands overwrite previous assignments; invalid IDs rejected. |
| **Playback State (Desired)** | Target state per display (`Playing`, `Paused`, `Stopped`, `Suspended`). | `ResourcePolicyEngine` (`core`) | `ProcessOrchestrator`, `StateCoordinator` | `ResourcePolicyEngine`, User UI toggle | Transient (In-Memory). | Occlusion/Battery policy overrides user "Play" request (Safety First). |
| **Playback State (Observed)**| Verified player presentation state (`SwapchainFrozen`, `DecodingActive`, `Error`). | `StateCoordinator` (`core`) | Gallery UI, Resource Policy Engine | Child Players via IPC ACK | Transient (In-Memory). | If player reports `Error`, state transitions immediately to `Failed`. |
| **Display Topology** | Physical monitor rects, scale factors (DPI), primary monitor flag, orientation. | `DisplayTopologyManager` (`core`) | All core modules, Gallery UI, Players | Win32 OS via `WM_DISPLAYCHANGE` | Transient (Re-queried on boot). | OS display geometry is authoritative; overrides stored configuration rects. |
| **Process State** | Spawned child PIDs, Job Object handles, crash history count, pipe session tokens. | `ProcessOrchestrator` (`core`) | Core Daemon modules | `ProcessOrchestrator` | Transient (In-Memory). | If crash count >3 in 60s, restart policy transitions to static fallback. |
| **Configuration & Library** | User settings, custom properties, wallpaper metadata cache, schema version. | `ConfigurationRepository` & `LibraryRepository` (`core`) | Gallery UI, Players | `ConfigurationRepository` | Persistent (`config.json`, `project.json`). | Atomic `.tmp` rename guarantees zero half-written files on crash. |
| **Audio Visualization** | WASAPI capture state (`Active`, `Idle`), 64-band FFT array, active subscribers. | `AudioCoordinator` (`core`) | Web Players via shared memory | `AudioCoordinator` | Transient (Volatile Memory Only). | Zero disk persistence; capture stops immediately when no web players active. |

---

## 3. Resource Ownership and Lifetime Matrix

The table below catalogs the 12 primary native and system resources, defining their creator, owner, borrowing rules, and deterministic cleanup mechanisms:

| Resource ID | Resource Entity | Creating Component | Owning Component | Permitted Borrowers | Expected Lifetime | Deterministic Cleanup Responsibility | Behavior on Process Crash |
|:---|:---|:---|:---|:---|:---|:---|:---|
| **`RES-01`** | **Player Processes** | `ProcessOrchestrator` (`core`) | `ProcessOrchestrator` (`core`) | Windows Job Object | Active duration of assigned wallpaper. | `ProcessOrchestrator` terminates via IPC or `TerminateProcess`. | Windows Job Object kills 100% of children automatically (`KILL_ON_JOB_CLOSE`). |
| **`RES-02`** | **Child Rendering HWNDs** | Child Player Process | Child Player Process | `DesktopHostManager` (reparents under `WorkerW`) | Process lifetime of child player. | Player calls `DestroyWindowW` on exit; core unlinks parent. | Win32 kernel cleans up window upon process death. |
| **`RES-03`** | **Desktop Shell HWNDs** | Windows Explorer | Windows Explorer (`Progman` / `WorkerW`) | `DesktopHostManager` (borrows for attachment) | Explorer session lifetime. | Restored to static wallpaper via `SystemParametersInfoW`. | Invalidated on Explorer crash; re-probed on `TaskbarCreated`. |
| **`RES-04`** | **Named Pipe Server** | `IpcServer` (`core`) | `IpcServer` (`core`) | Connected IPC Clients | Core daemon session lifetime. | `CloseHandle` called on daemon shutdown. | OS kernel closes pipe instance when core process terminates. |
| **`RES-05`** | **Windows Job Object** | `ProcessOrchestrator` (`core`) | `ProcessOrchestrator` (`core`) | Child Player Processes | Core daemon session lifetime. | Explicitly closed via `CloseHandle` during shutdown. | Kernel closes handle; kills all assigned child processes. |
| **`RES-06`** | **D3D11 Device & Swapchain**| Video/Web Player Process | Video/Web Player Process | DWM Desktop Compositor | Player playback session. | Released via COM `Release()` before window destruction. | D3D11 runtime destroys device context upon process exit. |
| **`RES-07`** | **`libmpv` Handle** | `kinetiwall-player-video` | `kinetiwall-player-video` | None | Player playback session. | `mpv_terminate_destroy()` invoked on exit. | OS frees allocated DLL memory upon process death. |
| **`RES-08`** | **WebView2 Controller** | `kinetiwall-player-web` | `kinetiwall-player-web` | None | Web player session. | `Close()` invoked on `ICoreWebView2Controller`. | Runtime cleans up renderer subprocesses upon host death. |
| **`RES-09`** | **WASAPI Audio Session** | `AudioCoordinator` (`core`) | `AudioCoordinator` (`core`) | None | Active visualization session. | `IAudioClient::Stop()` and COM release on shutdown. | Volatile ring buffer freed from heap. 0 bytes written to disk. |
| **`RES-10`** | **`config.json` File Locks**| `ConfigurationRepository` | `ConfigurationRepository` | None | Transient (Duration of atomic write). | File handle closed immediately after `sync_all()` and rename. | OS unlocks file upon controller death. |
| **`RES-11`** | **Wallpaper Media Files** | Filesystem Explorer | User / Local OS | Players (Read-Only) | Indefinite (User-managed). | None (User asset). Players open files with `FILE_SHARE_READ`. | File locks released by OS upon player exit. |
| **`RES-12`** | **Thumbnail Disk Cache** | `LibraryRepository` (`core`) | `LibraryRepository` (`core`) | Gallery UI (Read-Only) | Persisted until cache clean or asset deletion. | Old thumbnails purged when corresponding wallpaper deleted. | Re-generated on demand if deleted or corrupted. |

---

## 4. State Ownership Invariants & Non-Negotiable Rules

1. **The Gallery UI Is Never An Authority:** `kinetiwall-ui.exe` is strictly a view and command generator. It stores zero authoritative playback state. If the UI process is terminated abruptly via Task Manager, the core controller continues running without a single byte of state loss.
2. **No Direct Handle Sharing Across Processes:** Process and window handles (`HWND`, `HANDLE`) are never treated as plain integer tokens shared over Named Pipes without validation. Window reparenting is executed strictly through trusted win32 APIs (`SetParent`) within the controller after verifying that the target `HWND` belongs to the spawned child PID.
3. **Exclusive Persistence Authority:** Only `ConfigurationRepository` inside `kinetiwall-core.exe` is authorized to write to `config.json`. Child players and the Gallery UI are prohibited from directly opening configuration files for writing.

