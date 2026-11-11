# RECOVERY POLICY MATRIX (24 SCENARIOS)
## Document ID: KW-RCV-003 | Phase 3: Observability, Diagnostics & Operational Reliability

---

### 1. Authoritative Recovery Policy Matrix

This matrix specifies the exact recovery behaviors, retry constraints, fallback paths, and terminal outcomes across all 24 failure scenarios defined in Step 14:

```
+-------------------------------------------------------------------------------------------------------------------------+
|                                           RECOVERY POLICY MATRIX (POL-01 TO POL-24)                                     |
+--------+-----------------------+---------------------+-------------------+-----------------+----------------------------+
| POL ID | SCENARIO NAME         | COMPONENT & OWNER   | AUTO-RECOVERY?    | RETRY LIMIT     | FALLBACK ACTION            |
+--------+-----------------------+---------------------+-------------------+-----------------+----------------------------+
| POL-01 | Controller Boot Fail  | Core (Main Thread)  | NO (Fatal)        | 0 Retries       | Terminate, alert user      |
| POL-02 | UI Process Crash      | UI (Tauri Host)     | YES (On user req) | 0 Auto (Lazy)   | Core keeps playback running|
| POL-03 | Player Launch Fail    | Player (Supervisor) | YES (Transient)   | Max 1 Retry     | Stop monitor, notify user  |
| POL-04 | Renderer Init Timeout | Player (Supervisor) | YES (Transient)   | Max 1 Retry     | Kill PID, retry once       |
| POL-05 | Renderer Hard Crash   | Player (Supervisor) | YES (Transient)   | Max 3 in 60s    | Exponential backoff (1s-4s)|
| POL-06 | Renderer Freeze / Hang| Player (Supervisor) | YES (Transient)   | Max 2 in 60s    | Kill hung PID, respawn     |
| POL-07 | Invalid Wallpaper Pkg | PackageManager      | NO (Deterministic)| 0 Retries       | Reject import / assignment |
| POL-08 | Media File Missing    | Player / Core       | NO (Deterministic)| 0 Retries       | Detach display, alert user |
| POL-09 | IPC Pipe Timeout      | IPC Layer (Core/UI) | YES (Transient)   | Max 3 Retries   | Reconnect Named Pipe       |
| POL-10 | Child Exit Unexpected | Supervisor (Core)   | YES (Transient)   | Max 3 in 60s    | Triage exit code, respawn  |
| POL-11 | Crash Loop Tripped    | Supervisor (Core)   | NO (Exhausted)    | 0 Retries (Stop)| Trip circuit breaker       |
| POL-12 | Config Corrupt/Parse  | ConfigurationRepo   | YES (via .bak)    | Max 1 (.bak)    | Auto-restore config.bak    |
| POL-13 | Config Save Write Lock| ConfigurationRepo   | YES (Transient)   | Max 3 with 500ms| Flag unsaved, keep in RAM  |
| POL-14 | WebView2 Runtime Miss | Player-Web (Core)   | NO (Deterministic)| 0 Retries       | Disable web, offer runtime |
| POL-15 | HW Video Decode Fail  | Player-Video (mpv)  | YES (Graceful)    | 0 Retries       | Fallback to software CPU   |
| POL-16 | Display Topology Shift| MonitorManager      | YES (Reconcile)   | Debounce 500ms  | Rebind HWNDs, suspend lost |
| POL-17 | WorkerW Hook Failure  | DesktopHook (Core)  | YES (Alternative) | Max 1 Fallback  | Fall back to Progman hook  |
| POL-18 | Power Sleep & Resume  | Core State Manager  | YES (Revalidate)  | Bounded 1s      | Re-verify surfaces on wake |
| POL-19 | Log Storage Read-Only | DiagnosticsManager  | YES (Graceful)    | 0 Retries       | Fallback to memory ring buf|
| POL-20 | Low System RAM / VRAM | Core Supervisor     | YES (Throttle)    | 0 Retries       | Drop to 30fps / pause      |
| POL-21 | Shutdown Hang Timeout | Core Main Loop      | YES (Enforced)    | 250ms Hard Cap  | Force-kill via Job Object  |
| POL-22 | User Cancel In-Flight | Supervisor (Core)   | IMMEDIATE HALT    | 0 Retries       | Cancel backoff timer, stop |
| POL-23 | Rapid Switch Conflict | PlaybackCoordinator | SERIALIZED QUEUE  | Deduplicate     | Apply latest timestamp     |
| POL-24 | Exhausted Recover Retr| Supervisor (Core)   | USER-INITIATED    | 0 Auto          | Require explicit user click|
+--------+-----------------------+---------------------+-------------------+-----------------+----------------------------+
```

---

### 2. Exhaustive Scenario Specifications

#### POL-01: Controller Startup Failure
- **Trigger**: Win32 single-instance pipe collision or `%LOCALAPPDATA%` inaccessible.
- **Affected Component**: `kinetiwall-core.exe` | **Owner**: Main Entrypoint
- **Initial Severity**: `FATAL` | **Auto-Recovery**: `NO`
- **Action**: Check if running instance exists; if so, signal existing UI to focus and exit code `0`. If disk inaccessible, show native Win32 `MessageBoxW` and exit.
- **Terminal State**: `STOPPED` | **Diagnostic Event**: `APP_STARTUP_FAILED` (`APP_008`)
- **Traceability**: `KW-NF-006`, `SAT-001` | **Status**: **APPROVED**

#### POL-02: Gallery UI Process Failure
- **Trigger**: `kinetiwall-ui.exe` unhandled JavaScript error or native crash.
- **Affected Component**: `kinetiwall-ui.exe` | **Owner**: Tauri Supervisor
- **Initial Severity**: `WARN` | **Auto-Recovery**: `YES (Lazy on next user click)`
- **Action**: Log UI crash in Core; **do NOT restart UI automatically**. Wallpaper playback continues uninterrupted. UI respawns cleanly when the user clicks the tray icon.
- **Terminal State**: `NORMAL` | **Diagnostic Event**: `PLAYER_CRASH_DETECTED` (`PROC_005`)
- **Traceability**: `KW-NG-001`, `LIFE-004` | **Status**: **APPROVED**

#### POL-04: Renderer Initialization Timeout
- **Trigger**: Player subprocess fails to present first frame within $3,000\text{ ms}$.
- **Affected Component**: `kinetiwall-player-*` | **Owner**: `LifecycleSupervisor`
- **Initial Severity**: `ERROR` | **Auto-Recovery**: `YES` (Max 1 attempt)
- **Action**: Supervisor terminates unresponsive PID (`TerminateProcess`), waits $1,000\text{ ms}$, spawns fresh player instance. If second attempt times out, trip circuit breaker.
- **Terminal State**: `SAFE_STOP` (if exhausted) | **Diagnostic Event**: `PLAYER_STARTUP_TIMED_OUT` (`PROC_003`)
- **Traceability**: `KW-NF-012`, `FINJ-004` | **Status**: **APPROVED**

#### POL-05: Renderer Hard Crash (Access Violation 0xC0000005)
- **Trigger**: Subprocess dies with non-zero exit code during active playback.
- **Affected Component**: `kinetiwall-player-video.exe` | **Owner**: `LifecycleSupervisor`
- **Initial Severity**: `ERROR` | **Auto-Recovery**: `YES`
- **Retry Limit**: Max 3 crashes per 60-second sliding window.
- **Backoff Strategy**: Exponential: 1s (Attempt 1), 2s (Attempt 2), 4s (Attempt 3).
- **Terminal State**: `NORMAL` (if recovered) or `SAFE_STOP` (if 3 crashes exceeded).
- **Diagnostic Event**: `PLAYER_CRASH_DETECTED` $\to$ `RECOVERY_CYCLE_STARTED`.
- **Traceability**: `KW-NF-012`, `KW-MET-013`, `FINJ-001` | **Status**: **APPROVED**

#### POL-08: Wallpaper Media File Missing or Removed
- **Trigger**: User deletes video file from disk outside KinetiWall while engine is running.
- **Affected Component**: `PackageManager` / Player | **Owner**: `PlaybackCoordinator`
- **Initial Severity**: `ERROR` | **Auto-Recovery**: `NO` (Deterministic fault)
- **Action**: Zero retries. Detach display cleanly to native static desktop; flag wallpaper item in UI library with *"Missing File"* badge; emit non-modal toast alert.
- **Terminal State**: `SAFE_STOP` | **Diagnostic Event**: `WALLPAPER_SWITCH_FAILED` (`WALL_010`)
- **Traceability**: `KW-F-015`, `FINJ-008` | **Status**: **APPROVED**

#### POL-11: Repeated Child-Process Crashes (Circuit Breaker Tripped)
- **Trigger**: Fourth player crash within 60 seconds on the same display.
- **Affected Component**: All Players on Monitor | **Owner**: `LifecycleSupervisor`
- **Initial Severity**: `FATAL` (Scoped to monitor) | **Auto-Recovery**: `NO`
- **Action**: Permanent halt of automated retries. Trip circuit breaker. Detach desktop surface (`WorkerW`). Present user toast: *"Wallpaper on Display 1 crashed repeatedly and has been stopped [KW-ERR-511]"*.
- **Terminal State**: `SAFE_STOP` | **Diagnostic Event**: `SUPERVISOR_CRASH_LOOP_TRIPPED` (`PROC_010`)
- **Traceability**: `KW-NF-012`, `KW-R-002`, `FINJ-002` | **Status**: **APPROVED**

#### POL-12: Configuration File Corrupted or Unparseable
- **Trigger**: `config.json` fails `serde_json` deserialization on startup.
- **Affected Component**: `ConfigurationRepository` | **Owner**: Core Bootstrapper
- **Initial Severity**: `ERROR` | **Auto-Recovery**: `YES` (via `config.json.bak`)
- **Action**: Inspect `config.json.bak`. If valid, restore immediately over `config.json` and log `WARN_CONFIG_RESTORED_FROM_BAK`. If `.bak` also missing/invalid, archive corrupt file to `config.json.corrupt_{timestamp}`, write factory defaults, and alert user.
- **Terminal State**: `NORMAL` | **Diagnostic Event**: `CONFIG_WRITE_COMPLETED` (`STG_001`)
- **Traceability**: `KW-DAT-007`, `DEC-DATA-03`, `FINJ-012` | **Status**: **APPROVED**

#### POL-16: Display Topology Change (Monitor Detached / Attached)
- **Trigger**: Win32 `WM_DISPLAYCHANGE` notification received by Core message pump.
- **Affected Component**: `MonitorManager` | **Owner**: `PlaybackCoordinator`
- **Initial Severity**: `INFO` | **Auto-Recovery**: `YES`
- **Action**: Debounce display events for **500 ms** to let Windows display manager settle. Query `EnumDisplayMonitors`. Suspend players for detached monitors; spawn/reposition players for newly active displays based on EDID matching.
- **Terminal State**: `NORMAL` | **Diagnostic Event**: `DISPLAY_TOPOLOGY_CHANGED` (`DSK_003`)
- **Traceability**: `KW-F-020`, `ADR-0002`, `INT-002` | **Status**: **APPROVED**

#### POL-17: Desktop Host Initialization Failure (`WorkerW` Hook Failure)
- **Trigger**: Windows 11 internal shell changes prevent WorkerW creation via `0x052C`.
- **Affected Component**: Desktop Attachment Module | **Owner**: Core Desktop Hook
- **Initial Severity**: `ERROR` | **Auto-Recovery**: `YES` (Alternative fallback)
- **Action**: Attempt primary `WorkerW` parent lookup. If failed, fall back to secondary `Progman` child attachment (`DEC-PLT-001`). If both fail, do not force shell reparenting; notify user and offer borderless windowed mode. Zero Explorer crashes.
- **Terminal State**: `NORMAL` or `SAFE_STOP` | **Diagnostic Event**: `DESKTOP_INTEGRATION_FAILED` (`DSK_002`)
- **Traceability**: `KW-F-006`, `KW-R-004`, `FINJ-007` | **Status**: **APPROVED**

#### POL-21: Application Shutdown Timeout
- **Trigger**: Child player process hangs during graceful shutdown (`CMD_TERMINATE_PLAYER`).
- **Affected Component**: Core Teardown Loop | **Owner**: Core Process Supervisor
- **Initial Severity**: `WARN` | **Auto-Recovery**: `YES` (Forced kill)
- **Action**: Core waits maximum **250 ms** for child process exit. If PID remains alive, Core invokes Win32 `TerminateProcess(hProcess, 1)` and closes Job Object handle, guaranteeing total process termination.
- **Terminal State**: `STOPPED` | **Diagnostic Event**: `APP_SHUTDOWN_COMPLETED` (`APP_007`)
- **Traceability**: `KW-NF-006`, `DEC-PRF-08`, `FINJ-015` | **Status**: **APPROVED**

#### POL-22: User Cancellation During In-Flight Recovery
- **Trigger**: User clicks "Stop" in UI while player backoff timer is sleeping.
- **Affected Component**: `LifecycleSupervisor` | **Owner**: Core Task Coordinator
- **Initial Severity**: `INFO` | **Auto-Recovery**: `IMMEDIATE CANCELLATION`
- **Action**: Abort Tokio sleep timer via `CancellationToken`. Do not spawn replacement process. Detach display surface and transition directly to `SAFE_STOP`.
- **Terminal State**: `SAFE_STOP` | **Diagnostic Event**: `WALLPAPER_STOPPED` (`WALL_008`)
- **Traceability**: `KW-F-008`, `FINJ-010` | **Status**: **APPROVED**

