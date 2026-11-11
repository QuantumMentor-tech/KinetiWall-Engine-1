# Lifecycle Risk Register & Architectural Decisions
**Document ID:** `KW-LFC-006`  
**Deliverable:** F (Lifecycle Risk and Decision Log)  
**Parent Document:** [README.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/lifecycle_and_shutdown/README.md)  
**Classification:** Core System Architecture / Operational Reliability Standard  

---

## 1. Lifecycle & Teardown Risk Register

Managing processes and shell integration on Windows entails intricate OS edge cases. The following risk register identifies critical lifecycle failure modes, assesses impact, and defines mitigation architectures:

| Risk ID | Risk Summary & Failure Mode | Prob. | Impact | Severity | Architecture Mitigation & Prevention Strategy |
| :--- | :--- | :---: | :---: | :---: | :--- |
| **`RSK-LFC-01`** | **Orphan Player Renderers Post-Core Crash**<br>Core crashes or is killed externally; child renderers stay running invisibly beneath desktop icons consuming 100% GPU/CPU. | Med | High | **CRITICAL** | **Windows Job Object with `KILL_ON_JOB_CLOSE`**: Every child process is attached to Core's Job Object before its main thread is resumed. NT Kernel guarantees child termination when Core handle closes. |
| **`RSK-LFC-02`** | **Deadlock in Media Decoder Teardown**<br>`mpv_terminate_destroy()` or audio hardware buffer calls hang indefinitely during shutdown, blocking Core exit. | High | Med | **HIGH** | **Bounded Asynchronous Stop with Escalation**: Subprocess has $50\text{ ms}$ to cleanly close media instances; supervisor enforces an unyielding $250\text{ ms}$ total timeout before invoking `TerminateProcess`. |
| **`RSK-LFC-03`** | **Accidental Kill via OS PID Recycling**<br>Subprocess dies; OS assigns recycled PID to an unrelated app; Core issues kill command to innocent process. | Low | High | **CRITICAL** | **128-Bit Process UUIDs & Creation Timestamp Verification**: PIDs are never used as authoritative targets. Core verifies handle creation time (`GetProcessTimes`) before any management call. |
| **`RSK-LFC-04`** | **Corrupted Configuration on Hard Exit**<br>System reboot or user shutdown occurs while writing `config.json`, producing truncated/corrupt settings. | Med | Med | **HIGH** | **Atomic Staged Replace (`MoveFileExW`)**: Writes serialize to `config.json.tmp.<UUID>`, call `FlushFileBuffers`, then atomically swap via `MOVEFILE_WRITE_THROUGH`. |
| **`RSK-LFC-05`** | **Black Desktop / Missing Shell Wallpaper**<br>Player terminates or detaches from `WorkerW` without Windows shell restoring static wallpaper, leaving screen black. | Med | Med | **HIGH** | **Explicit Shell Restore Protocol**: Player calls `ShowWindow(hWnd, SW_HIDE)` and detaches parent window before releasing graphics pipeline, ensuring Windows shell wallpaper is visible. |
| **`RSK-LFC-06`** | **Ghost Notification Area Tray Icons**<br>Abrupt Core termination leaves orphaned tray icons in Windows Taskbar that disappear only when mouse hovers over them. | Med | Low | **MEDIUM** | **Explicit Cleanup in Stage 13 & Cold Re-scan**: Core calls `Shell_NotifyIconW(NIM_DELETE)` during Stage 13; installer/launcher sweeps tray on boot. |
| **`RSK-LFC-07`** | **Explorer Crash Breaks WorkerW Hierarchy**<br>`explorer.exe` crashes or restarts; `WorkerW` handle becomes invalid, causing players to render into orphaned window. | High | Med | **HIGH** | **`TaskbarCreated` Broadcast Interceptor**: Core listens for `TaskbarCreated`, re-spawns WorkerW via `0x052C`, and re-parents existing player HWNDs within $200\text{ ms}$. |
| **`RSK-LFC-08`** | **Crash & Restart Storm (Infinite Loop)**<br>Bad video codec or malformed web wallpaper crashes player repeatedly; supervisor restarts continuously, freezing system. | Med | High | **HIGH** | **Exponential Backoff & Circuit Breaker**: Maximum 3 restarts in a 60-second sliding window. Display falls back to static desktop wallpaper upon breaker trip. |

---

## 2. Architecture Decision Records (Lifecycle & Teardown: `DEC-LFC-01` to `08`)

The following formal decisions govern all lifecycle and resource management implementations across KinetiWall:

### `DEC-LFC-01`: Kernel Job Object Encapsulation for All Child Subprocesses
* **Context:** Video players and web renderers must never survive the Core supervisor process.
* **Decision:** All subprocesses spawned by `kinetiwall-core.exe` are created with `CREATE_SUSPENDED`, added to a Windows Job Object configured with `JOB_OBJECT_LIMIT_KILL_ON_JOB_CLOSE`, and only then resumed.
* **Consequences:** Provides kernel-level guarantee ($< 50\text{ ms}$) that no orphan process can survive Core termination, regardless of whether Core terminates cleanly, panics, or is killed via Task Manager.

### `DEC-LFC-02`: Ephemeral Lifecycle for Configuration and Library GUI (`kinetiwall-ui.exe`)
* **Context:** Desktop live wallpaper applications often maintain heavy Electron or Web-based GUI processes in the background, consuming $150\text{--}300\text{ MB}$ of persistent RAM.
* **Decision:** Closing the KinetiWall UI window terminates `kinetiwall-ui.exe` completely via `ExitProcess(0)`. The Core daemon handles all background supervision and tray interactions.
* **Consequences:** Ensures KinetiWall consumes exactly $0.0\text{ MB}$ of GUI memory when the settings window is closed, meeting requirement `KW-NG-001`. Active wallpaper playback continues uninterrupted.

### `DEC-LFC-03`: Dual-Key Process Identity (128-Bit UUID + Kernel Creation Timestamp)
* **Context:** Windows aggressively reuses 32-bit Process Identifiers (PIDs), introducing race conditions where a supervisor command could inadvertently terminate an unrelated application.
* **Decision:** Subprocesses are tracked internally using cryptographically random 128-bit UUIDs. Before executing any Win32 management API against an OS handle, Core checks the process start time (`GetProcessTimes`).
* **Consequences:** Completely eliminates PID recycling hazards. Management operations abort safely if handle metadata does not match recorded values.

### `DEC-LFC-04`: Fixed $250\text{ ms}$ Shutdown Budget with Unyielding Escalation
* **Context:** Windows user logoff and reboot sequences require rapid application exit. Slow cleanup processes cause Windows to display "This app is preventing shutdown".
* **Decision:** The cooperative shutdown phase is budgeted at a strict maximum of $250\text{ ms}$. Any subprocess that fails to terminate within this window is immediately terminated via `TerminateProcess(hProcess, 0xFE)`. Total shutdown must not exceed $500\text{ ms}$.
* **Consequences:** Guarantees compliance with `KW-NF-006`. Subsystems cannot stall application exit or system reboot.

### `DEC-LFC-05`: Atomic File Persistence via `MoveFileExW` During Teardown
* **Context:** Abrupt power loss or system shutdown during config writes can truncate `config.json`.
* **Decision:** In-memory settings and runtime state are written to temporary staging files, flushed via `FlushFileBuffers`, and replaced atomically using `MoveFileExW` with `MOVEFILE_WRITE_THROUGH | MOVEFILE_REPLACE_EXISTING`.
* **Consequences:** Prevents partial writes and file corruption. Unsuccessful writes are explicitly logged; no false success is reported.

### `DEC-LFC-06`: Quiescent State Entry on Session Lock and Monitor Standby
* **Context:** Rendering 60 FPS video while the workstation is locked or displays are sleeping wastes GPU/CPU cycles and drains battery.
* **Decision:** Core listens for `WTS_SESSION_LOCK`, `GUID_MONITOR_POWER_ON`, and `PBT_APMSUSPEND`. Upon receipt, Core signals players to freeze swapchain presentation, pause video decoders, and release audio endpoints.
* **Consequences:** Reduces background power consumption to near zero ($< 0.1\%$ CPU) when user is not viewing the desktop, satisfying `KW-F-008`.

### `DEC-LFC-07`: Safe Cold Boot Orphan Reconciliation
* **Context:** Unexpected host BSOD or developer debugger crashes could theoretically leave orphan processes if Job Objects failed to initialize.
* **Decision:** Core executes a non-destructive process sweep during cold startup, verifying target process executable image path hashes and command-line session flags before terminating matching stale processes.
* **Consequences:** Guarantees a clean desktop state upon boot while protecting all unrelated third-party applications.

### `DEC-LFC-08`: Explicit Shell Desktop Surface Restoration
* **Context:** Terminating a player without proper shell window restoration can leave a black desktop or frozen `WorkerW` surface.
* **Decision:** The player teardown protocol requires calling `ShowWindow(hWnd, SW_HIDE)` and `SetParent(hWnd, NULL)` prior to destroying the DirectX swapchain.
* **Consequences:** Exposes the native Windows static wallpaper seamlessly without screen flicker, shell restart, or `explorer.exe` interference.

---

## 3. Open Engineering Review Items

| Item ID | Description | Current Architectural Status | Target Phase |
| :---: | :--- | :--- | :---: |
| **`REV-LFC-01`** | **Dynamic Display Hotplug Teardown:** Evaluate potential GPU driver race conditions when a monitor is rapidly unplugged and plugged back in within $< 100\text{ ms}$. | Core debounces display change events with $500\text{ ms}$ hold-off timer. | Phase 4 (Runtime) |
| **`REV-LFC-02`** | **WebView2 Chromium Sandbox Teardown:** Verify whether child Chromium utility processes always terminate cleanly when parent Job Object is closed on Windows 10 vs Windows 11. | WebView2 Job Object limits confirmed in testing; will be validated on both OS builds. | Phase 4 (Runtime) |
| **`REV-LFC-03`** | **DirectX Device Loss on S3/S4 Sleep:** Assess swapchain recreation vs full player process restart upon waking from deep sleep. | Current specification restarts player process cleanly if `DXGI_ERROR_DEVICE_RESET` occurs. | Phase 4 (Runtime) |

