# Windows API Abstraction Architecture Specification
**Document ID:** `KW-FFI-001`  
**Deliverable:** A (Native Windows API Architecture)  
**Parent Document:** [README.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/native_api_abstraction/README.md)  
**Classification:** Windows Platform Integration / Architectural Specification  

---

## 1. Five-Tier Architectural Layering Model

To prevent fragile, platform-specific Win32 code from proliferating through higher-level engine modules, KinetiWall organizes all native Windows interactions into a strict five-tier architecture:

```
+---------------------------------------------------------------------------------+
| TIER 1: Application Domain Layer (Pure, Safe Rust)                               |
|   - Engine Lifecycle Controller, Wallpaper Orchestrator, Settings Manager       |
|   - Zero knowledge of HWND, HANDLE, Win32 error codes, or windows crate types   |
+---------------------------------------------------------------------------------+
                                      |
                                      v Uses Trait Interfaces
+---------------------------------------------------------------------------------+
| TIER 2: Platform Service Abstraction Layer (`kinetiwall_platform`)              |
|   - Traits: DisplayProvider, ShellIntegrator, ProcessSupervisor, PowerObserver  |
|   - Domain Types: DisplayRect, MonitorId, WindowGeometry, PowerEvent, ShellState|
|   - Mockable implementations for CI & Unit Testing                              |
+---------------------------------------------------------------------------------+
                                      |
                                      v Implemented by Safe Platform Adapters
+---------------------------------------------------------------------------------+
| TIER 3: Safe Native Wrappers & RAII Types (`kinetiwall_platform::win32`)        |
|   - Types: SafeHandle, SafeHwnd, SafeJobObject, MonitorSnapshot, WinEventSub     |
|   - Strict input validation, bounds checks, UTF-16 conversions, error mapping   |
|   - Zero exposed unsafe blocks; public API is 100% safe Rust                    |
+---------------------------------------------------------------------------------+
                                      |
                                      v Encapsulates Unsafe FFI Boundaries
+---------------------------------------------------------------------------------+
| TIER 4: Low-Level FFI Boundary (`kinetiwall_platform::win32::raw`)              |
|   - Narrow, audited unsafe blocks with mandatory // SAFETY: rationale comments  |
|   - Microsoft `windows` crate (v0.58) bindings                                  |
|   - Immediate GetLastError capture, pointer math validation, struct zeroing     |
+---------------------------------------------------------------------------------+
                                      |
                                      v Operating System ABI
+---------------------------------------------------------------------------------+
| TIER 5: Microsoft Windows Operating System (NT Kernel, User32, Shell32, DWM)   |
|   - Win32 APIs: CreateWindowExW, SetParent, EnumDisplayMonitors, Job Objects    |
+---------------------------------------------------------------------------------+
```

---

## 2. Native API Inventory & Responsibility Mapping

All Win32 operations required by KinetiWall are categorized into six functional modules:

### 2.1 Window Discovery & Shell Desktop Integration (`win32::shell`)
* **Operations:** Discover `Progman`, dispatch `0x052C` message, enumerate top-level `WorkerW` windows, validate behind-icon host, restore shell desktop background.
* **Consuming Components:** `kinetiwall-core.exe` (discovery coordinator), `kinetiwall-player-*.exe` (reparenting executor).
* **Native APIs:** `FindWindowW`, `SendMessageTimeoutW`, `EnumWindows`, `RealGetWindowClassW`, `GetWindowThreadProcessId`, `IsWindow`.
* **Classification:** **Mixed** (Win32 APIs are documented; `0x052C` message and `WorkerW` behind-icon split are **undocumented** shell behavior).

### 2.2 Window Management & Reparenting (`win32::window`)
* **Operations:** Create player render window, reparent into `WorkerW`, apply physical monitor bounds, configure transparent/click-through window styles, destroy window.
* **Consuming Components:** `kinetiwall-player-video.exe`, `kinetiwall-player-web.exe`.
* **Native APIs:** `CreateWindowExW`, `DestroyWindow`, `SetParent`, `SetWindowPos`, `ShowWindow`, `GetWindowLongPtrW`, `SetWindowLongPtrW`.
* **Classification:** **Documented Win32 User32 APIs**.

### 2.3 Display Enumeration & Virtual Desktop Geometry (`win32::display`)
* **Operations:** Enumerate physical and virtual monitors, query display bounds and refresh rates, obtain per-monitor DPI scaling, normalize negative virtual coordinates.
* **Consuming Components:** `kinetiwall-core.exe`, `kinetiwall-ui.exe`.
* **Native APIs:** `EnumDisplayMonitors`, `GetMonitorInfoW`, `GetSystemMetrics`, `GetDpiForMonitor`, `GetDpiForWindow`.
* **Classification:** **Documented Win32 GDI & DPI APIs**.

### 2.4 Process Supervision & Job Object Containment (`win32::process`)
* **Operations:** Create player child processes in suspended state, configure Job Object with `JOB_OBJECT_LIMIT_KILL_ON_JOB_CLOSE`, assign child to Job, resume thread, monitor process exit, escalate termination.
* **Consuming Components:** `kinetiwall-core.exe`.
* **Native APIs:** `CreateJobObjectW`, `SetInformationJobObject`, `AssignProcessToJobObject`, `CreateProcessW`, `ResumeThread`, `TerminateProcess`, `GetExitCodeProcess`, `WaitForMultipleObjects`.
* **Classification:** **Documented Win32 Kernel32 APIs**.

### 2.5 OS Event Notifications & Power Management (`win32::events`)
* **Operations:** Register for display topology changes, monitor sleep/wake events, workstation lock/unlock, and Explorer restarts.
* **Consuming Components:** `kinetiwall-core.exe`.
* **Native APIs:** `RegisterWindowMessageW`, `RegisterPowerSettingNotification`, `UnregisterPowerSettingNotification`, `WTSRegisterSessionNotification`, `WTSUnRegisterSessionNotification`, `SetWinEventHook`, `UnhookWinEvent`.
* **Classification:** **Documented Win32 User32 & Wtsapi32 APIs**.

### 2.6 Inter-Process Communication & Storage Synchronization (`win32::ipc_io`)
* **Operations:** Secure Named Pipe listeners, atomic file replace for settings, single-instance mutex.
* **Consuming Components:** All KinetiWall binaries.
* **Native APIs:** `CreateMutexW`, `ReleaseMutex`, `CreateNamedPipeW`, `ConnectNamedPipe`, `MoveFileExW`, `FlushFileBuffers`.
* **Classification:** **Documented Win32 Kernel32 APIs**.

---

## 3. Platform Trait Definitions (Mockable Tier 2 Interfaces)

Higher-level application services interact strictly through mockable Rust traits:

```rust
/// Trait representing display query and topology services
pub trait DisplayProvider: Send + Sync {
    /// Retrieves current active monitors with normalized virtual coordinates
    fn query_displays(&self) -> Result<Vec<DisplayDescriptor>, PlatformError>;
    
    /// Queries the DPI scaling factor for a specific display
    fn get_display_dpi(&self, display_id: &str) -> Result<u32, PlatformError>;
}

/// Trait representing Windows Shell integration and WorkerW discovery
pub trait ShellIntegrator: Send + Sync {
    /// Discovers the valid behind-icon WorkerW host window
    fn discover_wallpaper_host(&self) -> Result<ValidatedHostWindow, PlatformError>;
    
    /// Restores the native Windows desktop wallpaper surface
    fn restore_shell_wallpaper(&self) -> Result<(), PlatformError>;
}

/// Trait representing process supervision and orphan containment
pub trait ProcessSupervisor: Send + Sync {
    /// Spawns a supervised child player process enclosed in the Job Object
    fn spawn_player(&self, config: PlayerSpawnConfig) -> Result<SupervisedChild, PlatformError>;
    
    /// Terminates an active child process with an unyielding timeout
    fn terminate_child(&self, child_id: &Uuid, timeout_ms: u32) -> Result<(), PlatformError>;
}
```

---

## 4. Isolation of Undocumented Desktop-Host Mechanics

In strict conformance with `Principle K`, all code interacting with `Progman`, `WorkerW`, and message `0x052C` is quarantined inside `kinetiwall_platform::win32::shell::workerw_isolation`:

1. **Quarantine Boundary:** No public interface exposes raw `WorkerW` semantics to external modules. Higher-level code receives a typed, validated struct:
   ```rust
   #[derive(Debug, Clone)]
   pub struct ValidatedHostWindow {
       raw_hwnd: isize,
       owning_pid: u32,
       virtual_rect: DisplayRect,
       discovered_at: std::time::Instant,
   }
   ```
2. **Defensive Validation Gate:** A `ValidatedHostWindow` is only constructed if:
   * `IsWindow(hwnd)` is `TRUE`.
   * `RealGetWindowClassW(hwnd)` equals exactly `"WorkerW"`.
   * Owning PID matches the active `explorer.exe` process ID.
   * Target window does *not* contain `SHELLDLL_DefView` (preventing icon occlusion).
3. **Pluggable Architecture:** If a future Windows update modifies or deprecates `0x052C`, only the implementation of the `ShellIntegrator` trait requires modification; zero application or player code is affected.

---

## 5. Runtime Platform Capability Detection

To avoid scattering brittle `GetVersionEx` calls throughout the engine, KinetiWall uses a centralized, cached capability provider:

```rust
#[derive(Debug, Clone, Copy)]
pub struct PlatformCapabilities {
    /// True if Per-Monitor V2 DPI awareness is supported (Win10 1703+)
    pub supports_per_monitor_v2_dpi: bool,
    /// True if WorkerW split 0x052C is available and verified
    pub supports_workerw_desktop: bool,
    /// True if Direct3D 11 hardware decoding is available
    pub supports_d3d11_hardware_decode: bool,
    /// True if Windows Job Object kill-on-close is supported (NT 5.1+)
    pub supports_job_object_kill: bool,
}
```

Capability detection executes once during engine cold startup (`kinetiwall-core.exe` initialization) and is cached for the lifetime of the process. If a capability check fails (e.g. `supports_workerw_desktop == false`), KinetiWall safely disables live wallpaper hosting and transitions to `INTEGRATION_DEGRADED` without panicking.

