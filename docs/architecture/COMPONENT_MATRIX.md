# KinetiWall — Component Responsibility and Ownership Matrix
**Document ID:** `KW-ARC-003`  
**Deliverable:** Deliverable C — Component Responsibility and Ownership Matrix  
**Phase:** 2 — System Architecture & Technical Decisions  
**Step:** 7 of 90 — System Architecture & Architecture Decision Records  
**Status:** COMPLETE (Auditable Module Specifications)  
**Authority:** Principal Software Architect & Rust Systems Lead  
**Date:** 2026-11-10  

---

## 1. Modular Subsystem Overview

KinetiWall's core daemon (`kinetiwall-core.exe`) is partitioned into **10 cohesive, single-responsibility modules**. Each module possesses an unambiguous state boundary, an explicit event contract, and a deterministic failure-containment strategy.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                 10 CORE CONTROLLER ARCHITECTURAL MODULES                    │
├─────────────────────┬─────────────────────┬─────────────────────────────────┤
│ System Coordination │ Hardware & Shell    │ Data & Multimedia               │
├─────────────────────┼─────────────────────┼─────────────────────────────────┤
│ • AppLifecycleMgr   │ • DesktopHostMgr    │ • LibraryRepository             │
│ • ProcessOrchestr.  │ • DisplayTopologyMgr│ • ConfigurationRepo             │
│ • IpcServer & Auth  │ • ResourcePolicyEng │ • AudioCaptureEngine            │
│                     │ • PowerStateManager │ • DiagnosticsCoordinator        │
└─────────────────────┴─────────────────────┴─────────────────────────────────┘
```

---

## 2. Exhaustive Component Responsibility Specifications

### Component 1: `AppLifecycleManager`
- **Primary Responsibility:** Coordinates cold engine bootstrap, single-instance verification, Win32 system tray lifecycle, and clean application shutdown.
- **Owned State:** Engine run-state (`Bootstrapping`, `Running`, `ShuttingDown`), system tray icon handle (`NOTIFYICONDATAW`), single-instance mutex handle.
- **Public Contracts / Traits:** 
  ```rust
  pub trait LifecycleManager {
      fn initialize(&mut self) -> Result<(), LifecycleError>;
      fn run_event_loop(&mut self) -> Result<(), LifecycleError>;
      fn request_shutdown(&mut self, reason: ShutdownReason);
  }
  ```
- **Dependencies:** `ConfigurationRepository`, `DesktopHostManager`, `DisplayTopologyManager`, `IpcServer`.
- **Events Consumed:** Win32 session logon/logoff messages, Tray menu clicks (`ID_EXIT`, `ID_OPEN_GALLERY`, `ID_PAUSE`), `SIGTERM`.
- **Events / Commands Produced:** `EngineShutdownInitiated`, `OpenGalleryRequest`, `ToggleGlobalPause`.
- **Failure Handling:** If single-instance mutex exists, signals existing instance via Named Pipe and terminates immediately with exit code 0.
- **Testing Boundary:** Unit tests with mock Win32 message pump; headless bootstrap assertions.
- **Related Requirements:** `KW-F-005`, `KW-F-008`, `KW-NF-001`, `KW-NF-004`.

---

### Component 2: `ProcessOrchestrator`
- **Primary Responsibility:** Manages child wallpaper player processes inside Windows Job Objects; enforces bounded crash auto-restart with exponential backoff.
- **Owned State:** Map of active player processes (`DisplayId -> PlayerProcessRecord`), Windows Job Object handle (`HANDLE`), crash counter history per wallpaper.
- **Public Contracts / Traits:**
  ```rust
  pub trait ProcessSupervisor {
      fn spawn_player(&mut self, display: DisplayId, target: WallpaperTarget) -> Result<ProcessId, ProcessError>;
      fn terminate_player(&mut self, display: DisplayId) -> Result<(), ProcessError>;
      fn handle_child_exit(&mut self, pid: ProcessId, exit_code: u32);
      fn terminate_all_children(&mut self);
  }
  ```
- **Dependencies:** Windows Job Object APIs (`CreateJobObjectW`, `SetInformationJobObject`), Win32 `CreateProcessW`.
- **Events Consumed:** `ChildProcessTerminated` (via async thread pool wait on process handles).
- **Events / Commands Produced:** `PlayerSpawned`, `PlayerCrashed`, `PlayerRestartExhausted`, `StaticFallbackTriggered`.
- **Failure Handling:** Restarts crashed child player up to 3 times (delays: 100ms, 500ms, 1500ms). If crash count exceeds 3 in 60s, marks profile failed and triggers static desktop wallpaper.
- **Testing Boundary:** Mock process executor; automated fault-injection crash loops.
- **Related Requirements:** `KW-F-006`, `KW-F-007`, `KW-NF-010`.

---

### Component 3: `IpcServer` & Authorization Layer
- **Primary Responsibility:** Hosts local Windows Named Pipe; enforces interactive logon SID DACL; deserializes and authorizes JSON-RPC commands.
- **Owned State:** Named Pipe server handle (`HANDLE`), connected client sessions, message framing buffers (1 MB quota).
- **Public Contracts / Traits:**
  ```rust
  pub trait IpcGateway {
      fn start_listener(&mut self, pipe_name: &str) -> Result<(), IpcError>;
      fn broadcast_event(&mut self, event: &EngineEvent) -> Result<(), IpcError>;
      fn dispatch_command(&mut self, msg: IpcMessage) -> IpcResponse;
  }
  ```
- **Dependencies:** Win32 Named Pipe APIs, Win32 Advapi32 Security Descriptors, `serde_json`.
- **Events Consumed:** Incoming binary streams on `\\.\pipe\kinetiwall-ipc-{SessionId}`.
- **Events / Commands Produced:** `ApplyWallpaperCommand`, `UpdatePropertyCommand`, `QueryStateRequest`.
- **Failure Handling:** Drops malformed or oversized payloads (>1 MB); resets client pipe state on unexpected disconnect without blocking listener.
- **Testing Boundary:** Fuzz testing with corrupted JSON, oversized buffers, and secondary user token connection tests.
- **Related Requirements:** `KW-NF-003`, `KW-NF-012`, `KW-NF-013`.

---

### Component 4: `DesktopHostManager`
- **Primary Responsibility:** Encapsulates Windows shell integration: broadcasts `0x052C` to `Progman`, locates `WorkerW`, executes dual-mode attachment fallback, and recovers on Explorer crash.
- **Owned State:** Current `Progman` HWND, active `WorkerW` HWND, desktop icon visibility state, registered `TaskbarCreated` message ID.
- **Public Contracts / Traits:**
  ```rust
  pub trait DesktopHost {
      fn attach_surface(&mut self, child_hwnd: HWND, rect: Rect) -> Result<(), AttachmentError>;
      fn detach_surface(&mut self, child_hwnd: HWND) -> Result<(), AttachmentError>;
      fn handle_explorer_restart(&mut self) -> Result<(), AttachmentError>;
      fn restore_static_desktop(&mut self) -> Result<(), AttachmentError>;
  }
  ```
- **Dependencies:** Win32 User32 (`SendMessageTimeoutW`, `FindWindowW`, `SetParent`, `SystemParametersInfoW`).
- **Events Consumed:** `TaskbarCreated` window message, Desktop icon visibility toggle notifications.
- **Events / Commands Produced:** `DesktopHostReady`, `DesktopHostInvalidated`, `ShellAttachmentRestored`.
- **Failure Handling:** If `WorkerW` is unavailable (e.g. Win 11 24H2 hidden icons), executes fallback anchoring directly beneath `SHELLDLL_DefView` (`DEC-PLT-001`).
- **Testing Boundary:** Mock window hierarchy inspector; integration tests across Windows 11 24H2 and Windows 10 22H2.
- **Related Requirements:** `KW-F-001`, `KW-F-002`, `KW-F-003`, `KW-F-004`.

---

### Component 5: `DisplayTopologyManager`
- **Primary Responsibility:** Discovers active physical monitors via `EnumDisplayMonitors`, manages mixed-DPI scaling, and tracks virtual screen coordinates.
- **Owned State:** Active display registry (`DisplayId -> DisplayInfo { rect, dpi, is_primary, orientation }`), active topology mode (`Independent`, `Span`, `Clone`).
- **Public Contracts / Traits:**
  ```rust
  pub trait TopologyManager {
      fn enumerate_displays(&mut self) -> Result<Vec<DisplayInfo>, TopologyError>;
      fn get_display_for_rect(&self, rect: Rect) -> Option<DisplayId>;
      fn handle_display_change(&mut self) -> TopologyDelta;
  }
  ```
- **Dependencies:** Win32 User32 (`EnumDisplayMonitors`, `GetMonitorInfoW`, `GetDpiForMonitor`).
- **Events Consumed:** `WM_DISPLAYCHANGE`, `WM_DPICHANGED`.
- **Events / Commands Produced:** `DisplayConnected`, `DisplayDisconnected`, `DisplayDpiChanged`.
- **Failure Handling:** Re-queries display geometry on any topology message; migrates orphaned wallpapers to primary display if an assigned monitor disappears.
- **Testing Boundary:** Simulated multi-monitor virtual environments and display disconnect harnesses.
- **Related Requirements:** `KW-F-020` through `KW-F-025`.

---

### Component 6: `ResourcePolicyEngine`
- **Primary Responsibility:** Tracks foreground windows and occlusion via Win32 event hooks; evaluates pause/resume rules for gaming and battery conservation.
- **Owned State:** Foreground window handle (`HWND`), foreground window rect, per-monitor occlusion map, global pause override toggle.
- **Public Contracts / Traits:**
  ```rust
  pub trait PolicyEngine {
      fn register_hooks(&mut self) -> Result<(), PolicyError>;
      fn evaluate_occlusion(&mut self, fg_hwnd: HWND) -> OcclusionAction;
      fn should_pause_display(&self, display: DisplayId) -> bool;
  }
  ```
- **Dependencies:** Win32 User32 (`SetWinEventHook`, `GetWindowRect`, `GetWindowLongPtrW`).
- **Events Consumed:** `EVENT_SYSTEM_FOREGROUND`, `EVENT_OBJECT_LOCATIONCHANGE`.
- **Events / Commands Produced:** `PauseDisplayPlayback`, `ResumeDisplayPlayback`.
- **Failure Handling:** Ignores transparent tooltips and invisible windows; defaults to unpaused playback if window inspection fails.
- **Testing Boundary:** Automated test harness opening fullscreen and minimized windows; CapFrameX gaming impact tests.
- **Related Requirements:** `KW-F-026`, `KW-F-027`, `KW-F-028`, `KW-F-030`, `KW-NF-004`, `KW-NF-005`.

---

### Component 7: `PowerStateManager`
- **Primary Responsibility:** Monitors system AC vs battery power states and battery saver mode; signals power-saving policies.
- **Owned State:** Current power source (`AC`, `Battery`), battery percentage, battery saver active flag.
- **Public Contracts / Traits:**
  ```rust
  pub trait PowerMonitor {
      fn query_power_status(&self) -> Result<PowerStatus, PowerError>;
      fn register_notifications(&mut self, hwnd: HWND) -> Result<(), PowerError>;
  }
  ```
- **Dependencies:** Win32 Kernel32 (`GetSystemPowerStatus`), `RegisterPowerSettingNotification`.
- **Events Consumed:** `WM_POWERBROADCAST` (`PBT_APMPOWERSTATUSCHANGE`, `PBT_POWERSETTINGCHANGE`).
- **Events / Commands Produced:** `PowerSourceChanged(Battery)`, `PowerSourceChanged(AC)`.
- **Failure Handling:** If power queries fail, assumes AC power to prevent unintended wallpaper pausing.
- **Testing Boundary:** Simulated Win32 power broadcast messages.
- **Related Requirements:** `KW-F-029`, `KW-F-031`.

---

### Component 8: `LibraryRepository`
- **Primary Responsibility:** Indexes local wallpaper folders, validates `project.json` manifests, extracts ZIP archives with Zip-Slip protection, and caches thumbnails.
- **Owned State:** In-memory wallpaper catalog (`WallpaperId -> WallpaperMetadata`), thumbnail disk cache index.
- **Public Contracts / Traits:**
  ```rust
  pub trait WallpaperLibrary {
      fn scan_library(&mut self) -> Result<Vec<WallpaperMetadata>, LibraryError>;
      fn import_archive(&mut self, archive_path: &Path) -> Result<WallpaperId, ImportError>;
      fn get_metadata(&self, id: WallpaperId) -> Option<&WallpaperMetadata>;
  }
  ```
- **Dependencies:** Rust `std::fs`, `serde_json`, `zip` crate.
- **Events Consumed:** `ImportWallpaperRequest`, `DeleteWallpaperRequest`.
- **Events / Commands Produced:** `LibraryCatalogUpdated`, `WallpaperImportCompleted`.
- **Failure Handling:** Rejects archives with traversal paths (`../`); quarantines corrupt manifests without failing library scan.
- **Testing Boundary:** Unit tests parsing synthetic malformed, truncated, and valid `project.json` files.
- **Related Requirements:** `KW-F-032` through `KW-F-036`, `KW-NF-014`.

---

### Component 9: `ConfigurationRepository`
- **Primary Responsibility:** Manages persistent user settings, active wallpaper assignments, and custom properties with transactional atomic file writes.
- **Owned State:** In-memory `EngineConfig` struct, dirty state flag, path to `config.json`.
- **Public Contracts / Traits:**
  ```rust
  pub trait ConfigRepository {
      fn load(&mut self) -> Result<EngineConfig, ConfigError>;
      fn save(&mut self, config: &EngineConfig) -> Result<(), ConfigError>;
      fn restore_backup(&mut self) -> Result<EngineConfig, ConfigError>;
  }
  ```
- **Dependencies:** Rust `std::fs`, `tempfile` crate, `serde_json`.
- **Events Consumed:** `UpdateConfigCommand`, Engine shutdown signal.
- **Events / Commands Produced:** `ConfigSaved`, `ConfigCorruptedFallback`.
- **Failure Handling:** Writes atomically to `config.json.tmp`, flushes, and renames over `config.json`. If primary is corrupt, auto-restores from `config.json.bak`.
- **Testing Boundary:** Simulated power-kill write tests and corrupt JSON recovery tests.
- **Related Requirements:** `KW-F-037`.

---

### Component 10: `AudioCaptureEngine`
- **Primary Responsibility:** Captures system multimedia loopback audio via WASAPI; calculates 1024-point SIMD FFT in pure Rust; streams frequency bands to web players.
- **Owned State:** WASAPI `IAudioClient` handle, volatile ring buffer, FFT calculator instance (`realfft`).
- **Public Contracts / Traits:**
  ```rust
  pub trait AudioVisualizer {
      fn start_capture(&mut self) -> Result<(), AudioError>;
      fn stop_capture(&mut self);
      fn get_latest_frequency_bands(&self) -> [f32; 64];
  }
  ```
- **Dependencies:** Windows Core Audio WASAPI (`MMDeviceEnumerator`, `IAudioClient`), Rust `realfft`.
- **Events Consumed:** Audio endpoint change (`WM_DEVICECHANGE`), Audio visualizer enable/disable toggles.
- **Events / Commands Produced:** `AudioSpectrumFrameAvailable` (60 FPS).
- **Failure Handling:** Audio data exists strictly in volatile RAM (0 bytes written to disk); auto-rebinds capture client in <500ms on device switch.
- **Testing Boundary:** 1 kHz sine wave accuracy assertions; Process Monitor 0-byte disk write audit.
- **Related Requirements:** `KW-F-039` through `KW-F-042`, `KW-NF-014`, `KW-NF-015`, `KW-NG-005`.

