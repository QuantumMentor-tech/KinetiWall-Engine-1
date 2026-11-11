# KinetiWall — In-Process Module Responsibility Matrix
**Document ID:** `KW-PRC-003`  
**Deliverable:** Deliverable C — In-Process Module Responsibility Matrix  
**Phase:** 2 — System Architecture & Technical Decisions  
**Step:** 8 of 90 — Multi-Process Boundaries & Component Responsibilities  
**Status:** COMPLETE (Auditable Module Architecture Baseline)  
**Authority:** Principal Software Architect & IPC Systems Lead  
**Date:** 2026-11-10  

---

## 1. Architectural Justification for In-Process Modules

A common architectural anti-pattern in desktop software is **over-modularization into micro-processes**. Creating separate executables for configuration management, display topology, or power monitoring would introduce:
1. Significant IPC serialization overhead and context-switching penalties.
2. Fragile multi-process synchronization and distributed state hazards.
3. Substantial memory bloat (each separate Rust process allocates 8–12 MB for runtime/allocator structures).

Therefore, KinetiWall enforces a single, cohesive, multi-threaded executable for the core controller (`kinetiwall-core.exe`), partitioning internal logic into **12 single-responsibility in-process modules**:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                 12 CORE CONTROLLER IN-PROCESS MODULES                       │
├─────────────────────┬─────────────────────┬─────────────────────────────────┤
│ Lifecycle & Process │ Hardware & Shell    │ State, Data & Audio             │
├─────────────────────┼─────────────────────┼─────────────────────────────────┤
│ • AppLifecycleMgr   │ • DesktopHostMgr    │ • StateCoordinator              │
│ • ProcessOrchestr.  │ • DisplayTopologyMgr│ • WallpaperAssignmentMgr        │
│ • IpcServer & Auth  │ • ResourcePolicyEng │ • ConfigurationRepo             │
│                     │ • PowerStateManager │ • DiagnosticsCoordinator        │
│                     │                     │ • AudioCoordinator (WASAPI FFT) │
└─────────────────────┴─────────────────────┴─────────────────────────────────┘
```

---

## 2. Exhaustive In-Process Module Specifications

### Module 1: `AppLifecycleManager`
- **Primary Purpose:** Orchestrates cold engine bootstrap, single-instance verification, tray icon creation, and clean shutdown.
- **Owned State:** Engine run-state (`Bootstrapping`, `Running`, `ShuttingDown`), system tray icon handle (`NOTIFYICONDATAW`), single-instance mutex.
- **Public Interface:** `initialize()`, `run_event_loop()`, `request_shutdown(reason)`.
- **Excluded Responsibilities:** Does not spawn player processes directly (delegates to `ProcessOrchestrator`).
- **Dependencies:** `ConfigurationRepository`, `DesktopHostManager`, `DisplayTopologyManager`, `IpcServer`.
- **Events Consumed:** Win32 session logon/logoff messages, tray clicks (`ID_EXIT`, `ID_OPEN_GALLERY`).
- **Events Emitted:** `EngineShutdownInitiated`, `OpenGalleryRequest`.
- **Resource Ownership:** Win32 system tray icon, single-instance named mutex.
- **Failure Behavior:** If mutex exists, signals existing instance via pipe and exits cleanly with 0.
- **Test Boundary:** Unit tests with mock Win32 message pump; headless bootstrap assertions.
- **Related Requirements:** `KW-F-005`, `KW-F-008`, `KW-NF-001`, `KW-NF-004`.
- **Rationale for In-Process:** Must run in the primary daemon thread to host the Win32 message loop required for `Shell_NotifyIconW`.

---

### Module 2: `ProcessOrchestrator`
- **Primary Purpose:** Spawns and supervises out-of-process wallpaper players inside Windows Job Objects; enforces bounded crash auto-restart.
- **Owned State:** Child process table (`DisplayId -> PlayerProcessRecord`), Job Object handle, crash history map.
- **Public Interface:** `spawn_player(display, target)`, `terminate_player(display)`, `handle_child_exit(pid, code)`, `terminate_all()`.
- **Excluded Responsibilities:** Does not decode video or execute web scripts (delegates to child executables).
- **Dependencies:** Win32 Job Objects (`CreateJobObjectW`), `CreateProcessW`.
- **Events Consumed:** `ChildProcessTerminated` (via async thread wait).
- **Events Emitted:** `PlayerSpawned`, `PlayerCrashed`, `PlayerRestartExhausted`, `StaticFallbackTriggered`.
- **Resource Ownership:** Windows Job Object handle, child process handles.
- **Failure Behavior:** Applies exponential backoff (100ms, 500ms, 1500ms); halts restarts after 3 failures in 60s.
- **Test Boundary:** Mock process executor; automated fault-injection crash loops.
- **Related Requirements:** `KW-F-006`, `KW-F-007`, `KW-NF-010`.
- **Rationale for In-Process:** Must reside in the controller to maintain the Job Object handle that guarantees child cleanup upon daemon termination.

---

### Module 3: `IpcServer` & Authorization Layer
- **Primary Purpose:** Hosts local Windows Named Pipe; enforces interactive logon SID DACL; deserializes and authorizes JSON-RPC commands.
- **Owned State:** Named Pipe server handle, connected client sessions, message framing buffers (1 MB quota).
- **Public Interface:** `start_listener(pipe_name)`, `broadcast_event(event)`, `dispatch_command(msg)`.
- **Excluded Responsibilities:** Does not execute business logic directly (routes commands to responsible modules).
- **Dependencies:** Win32 Named Pipe APIs, Win32 Advapi32 Security Descriptors, `serde_json`.
- **Events Consumed:** Incoming binary streams on `\\.\pipe\kinetiwall-ipc-{SessionId}`.
- **Events Emitted:** `ApplyWallpaperCommand`, `UpdatePropertyCommand`, `QueryStateRequest`.
- **Resource Ownership:** Named Pipe handle, client session tokens.
- **Failure Behavior:** Drops malformed/oversized payloads (>1 MB); resets client pipe state on disconnect without blocking listener.
- **Test Boundary:** Fuzz testing with corrupted JSON, oversized buffers, and foreign token connection tests.
- **Related Requirements:** `KW-NF-003`, `KW-NF-012`, `KW-NF-013`.
- **Rationale for In-Process:** Direct in-memory dispatch to controller modules eliminates multi-hop IPC latency.

---

### Module 4: `DesktopHostManager`
- **Primary Purpose:** Encapsulates Windows shell integration: broadcasts `0x052C` to `Progman`, locates `WorkerW`, executes dual-mode attachment fallback, and recovers on Explorer crash.
- **Owned State:** Current `Progman` HWND, active `WorkerW` HWND, desktop icon visibility state, registered `TaskbarCreated` message ID.
- **Public Interface:** `attach_surface(child_hwnd, rect)`, `detach_surface(child_hwnd)`, `handle_explorer_restart()`, `restore_static_desktop()`.
- **Excluded Responsibilities:** Does not manage playback states or video decoding.
- **Dependencies:** Win32 User32 (`SendMessageTimeoutW`, `FindWindowW`, `SetParent`, `SystemParametersInfoW`).
- **Events Consumed:** `TaskbarCreated` window message, desktop icon visibility toggle notifications.
- **Events Emitted:** `DesktopHostReady`, `DesktopHostInvalidated`, `ShellAttachmentRestored`.
- **Resource Ownership:** Parent shell HWND references.
- **Failure Behavior:** If `WorkerW` is destroyed (Win 11 24H2 hidden icons), engages fallback anchoring beneath `Progman` child hierarchy (`DEC-PLT-001`).
- **Test Boundary:** Mock window hierarchy inspector; integration tests across Windows 11 24H2 and Windows 10 22H2.
- **Related Requirements:** `KW-F-001`, `KW-F-002`, `KW-F-003`, `KW-F-004`.
- **Rationale for In-Process:** Requires high-frequency win32 window message processing in the interactive user session.

---

### Module 5: `DisplayTopologyManager`
- **Primary Purpose:** Discovers active physical monitors via `EnumDisplayMonitors`, manages mixed-DPI scaling, and tracks virtual screen coordinates.
- **Owned State:** Active display registry (`DisplayId -> DisplayInfo { rect, dpi, is_primary, orientation }`), active topology mode (`Independent`, `Span`, `Clone`).
- **Public Interface:** `enumerate_displays()`, `get_display_for_rect(rect)`, `handle_display_change()`.
- **Excluded Responsibilities:** Does not own wallpaper profile persistence.
- **Dependencies:** Win32 User32 (`EnumDisplayMonitors`, `GetMonitorInfoW`, `GetDpiForMonitor`).
- **Events Consumed:** `WM_DISPLAYCHANGE`, `WM_DPICHANGED`.
- **Events Emitted:** `DisplayConnected`, `DisplayDisconnected`, `DisplayDpiChanged`.
- **Resource Ownership:** Display geometry tables.
- **Failure Behavior:** Re-queries display geometry on any topology message; migrates orphaned wallpapers to primary display if an assigned monitor disappears.
- **Test Boundary:** Simulated multi-monitor virtual environments and display disconnect harnesses.
- **Related Requirements:** `KW-F-020` through `KW-F-025`.
- **Rationale for In-Process:** Must coordinate directly with `DesktopHostManager` and `ProcessOrchestrator` without inter-process latency.

---

### Module 6: `ResourcePolicyEngine`
- **Primary Purpose:** Tracks foreground windows and occlusion via Win32 event hooks; evaluates pause/resume rules for gaming and battery conservation.
- **Owned State:** Foreground window handle (`HWND`), foreground window rect, per-monitor occlusion map, global pause override toggle.
- **Public Interface:** `register_hooks()`, `evaluate_occlusion(fg_hwnd)`, `should_pause_display(display)`.
- **Excluded Responsibilities:** Does not implement video decoding presentation.
- **Dependencies:** Win32 User32 (`SetWinEventHook`, `GetWindowRect`, `GetWindowLongPtrW`).
- **Events Consumed:** `EVENT_SYSTEM_FOREGROUND`, `EVENT_OBJECT_LOCATIONCHANGE`.
- **Events Emitted:** `PauseDisplayPlayback`, `ResumeDisplayPlayback`.
- **Resource Ownership:** WinEvent hook handles (`HWINEVENTHOOK`).
- **Failure Behavior:** Ignores transparent tooltips and invisible windows; debounces rapid Alt+Tabs by 50ms.
- **Test Boundary:** Automated test harness opening fullscreen and minimized windows; CapFrameX gaming impact tests.
- **Related Requirements:** `KW-F-026`, `KW-F-027`, `KW-F-028`, `KW-F-030`, `KW-NF-004`, `KW-NF-005`.
- **Rationale for In-Process:** Hook callbacks execute in the context of the controller thread; sub-50ms latency requires zero IPC hops to evaluate.

---

### Module 7: `PowerStateManager`
- **Primary Purpose:** Monitors system AC vs battery power states and battery saver mode; signals power-saving policies.
- **Owned State:** Current power source (`AC`, `Battery`), battery percentage, battery saver active flag.
- **Public Interface:** `query_power_status()`, `register_notifications(hwnd)`.
- **Excluded Responsibilities:** Does not pause players directly (signals `ResourcePolicyEngine`).
- **Dependencies:** Win32 Kernel32 (`GetSystemPowerStatus`), `RegisterPowerSettingNotification`.
- **Events Consumed:** `WM_POWERBROADCAST` (`PBT_APMPOWERSTATUSCHANGE`, `PBT_POWERSETTINGCHANGE`).
- **Events Emitted:** `PowerSourceChanged(Battery)`, `PowerSourceChanged(AC)`.
- **Resource Ownership:** Power notification registration handles.
- **Failure Behavior:** If power queries fail, assumes AC power to prevent unintended wallpaper pausing.
- **Test Boundary:** Simulated Win32 power broadcast messages.
- **Related Requirements:** `KW-F-029`, `KW-F-031`.
- **Rationale for In-Process:** Tightly coupled with the core daemon's message pump.

---

### Module 8: `WallpaperAssignmentManager`
- **Primary Purpose:** Maintains the authoritative mapping between physical display IDs and assigned wallpaper identities across Independent, Span, and Clone modes.
- **Owned State:** Map of `DisplayId -> AssignedWallpaperProfile`.
- **Public Interface:** `assign_wallpaper(display, profile)`, `get_assignment(display)`, `remove_assignment(display)`.
- **Excluded Responsibilities:** Does not spawn processes or write to disk.
- **Dependencies:** `DisplayTopologyManager`.
- **Events Consumed:** `ApplyWallpaperCommand`.
- **Events Emitted:** `AssignmentUpdated`.
- **Resource Ownership:** In-memory assignment table.
- **Failure Behavior:** Re-evaluates assignments when displays are added or removed.
- **Test Boundary:** Multi-monitor assignment logic unit tests.
- **Related Requirements:** `KW-F-021`, `KW-F-022`, `KW-F-023`.
- **Rationale for In-Process:** Core state coordination module; keeping it in-process avoids distributed state desynchronization.

---

### Module 9: `StateCoordinator`
- **Primary Purpose:** Reconciles the *Desired Playback State* (requested by user/policy) with the *Observed Playback State* (reported by player processes).
- **Owned State:** Master state table per display (`DesiredState`, `ObservedState`, `StateTransitionTimestamp`).
- **Public Interface:** `request_state(display, state)`, `report_observed_state(display, state)`, `get_display_state(display)`.
- **Excluded Responsibilities:** Does not execute win32 hooks.
- **Dependencies:** None (Pure logic engine).
- **Events Consumed:** State change requests from UI/policy; status reports from players.
- **Events Emitted:** `StateReconciliationError` (if observed state fails to match desired state within 2.0s).
- **Resource Ownership:** Formal state machine records.
- **Failure Behavior:** Marks display state as `Recovering` if player fails to acknowledge state transition.
- **Test Boundary:** Deterministic state machine transition unit tests.
- **Related Requirements:** `KW-AC-016`, `KW-MET-009`.
- **Rationale for In-Process:** Foundational invariant: prevents UI from falsely believing a player is paused when it is still rendering.

---

### Module 10: `ConfigurationRepository`
- **Primary Purpose:** Manages persistent user settings, active wallpaper profiles, and custom property states with transactional atomic file writes.
- **Owned State:** In-memory `EngineConfig` struct, dirty state flag, path to `config.json`.
- **Public Interface:** `load()`, `save(config)`, `restore_backup()`.
- **Excluded Responsibilities:** Does not parse wallpaper archive packages.
- **Dependencies:** Rust `std::fs`, `tempfile` crate, `serde_json`.
- **Events Consumed:** `UpdateConfigCommand`, Engine shutdown signal.
- **Events Emitted:** `ConfigSaved`, `ConfigCorruptedFallback`.
- **Resource Ownership:** File locks on `config.json`.
- **Failure Behavior:** Writes atomically to `config.json.tmp`, flushes, and renames over `config.json`. If primary is corrupt, auto-restores from `config.json.bak`.
- **Test Boundary:** Simulated power-kill write tests and corrupt JSON recovery tests.
- **Related Requirements:** `KW-F-037`.
- **Rationale for In-Process:** Avoids external database daemon overhead; pure Rust atomic file I/O takes <1ms.

---

### Module 11: `DiagnosticsCoordinator`
- **Primary Purpose:** Aggregates structured telemetry, logs engine events, records crash diagnostics, and sanitizes private data.
- **Owned State:** Rolling log buffer (10 MB ceiling), diagnostic session ID.
- **Public Interface:** `log_event(level, component, msg)`, `record_crash(dump_info)`, `flush_logs()`.
- **Excluded Responsibilities:** Does not transmit telemetry over the network (`KW-NG-002`).
- **Dependencies:** Rust `tracing` / `tracing-subscriber`.
- **Events Consumed:** Log events from all core modules.
- **Events Emitted:** None (Local disk writer only).
- **Resource Ownership:** Log file handles in `%LOCALAPPDATA%\KinetiWall\logs\`.
- **Failure Behavior:** Discards excess log lines if disk write buffer overflows; enforces rolling 10 MB file ceiling.
- **Test Boundary:** Log sanitization assertions confirming 0 personal paths or passwords recorded.
- **Related Requirements:** `KW-NG-002`, `KW-NF-021`.
- **Rationale for In-Process:** Zero-overhead tracing instrumentation throughout all daemon modules.

---

### Module 12: `AudioCoordinator` (In-Process WASAPI Subsystem)
- **Primary Purpose:** Captures multimedia loopback PCM audio via WASAPI, calculates 1024-point SIMD FFT in pure Rust, and streams 64 frequency bands to active web players.
- **Owned State:** WASAPI `IAudioClient` handle, volatile ring buffer, FFT calculator instance (`realfft`).
- **Public Interface:** `start_capture()`, `stop_capture()`, `get_latest_spectrum()`.
- **Excluded Responsibilities:** Does not serialize audio data to disk or stream across networks (`KW-NG-005`).
- **Dependencies:** Windows Core Audio WASAPI (`MMDeviceEnumerator`, `IAudioClient`), Rust `realfft`.
- **Events Consumed:** Audio endpoint changes (`WM_DEVICECHANGE`), audio visualizer enable/disable toggles.
- **Events Emitted:** `AudioSpectrumFrameAvailable` (60 FPS).
- **Resource Ownership:** WASAPI COM client handles, volatile shared memory mapping.
- **Failure Behavior:** Audio data exists strictly in volatile RAM; auto-rebinds capture client in <500ms on device switch.
- **Test Boundary:** 1 kHz sine wave accuracy assertions; Process Monitor 0-byte disk write audit.
- **Related Requirements:** `KW-F-039`..`042`, `KW-NF-014`, `KW-NF-015`, `KW-NG-005`.
- **Rationale for In-Process:** WASAPI loopback capture is ultra-low CPU (<0.3%); hosting it in `core` avoids an extra audio process while keeping memory volatile.

