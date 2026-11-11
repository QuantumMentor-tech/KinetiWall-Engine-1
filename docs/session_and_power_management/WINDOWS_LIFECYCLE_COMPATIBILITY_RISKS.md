# Windows Lifecycle Compatibility Risks & Decisions
**Document ID:** `KW-LCE-006`  
**Deliverable:** F (Windows Lifecycle Compatibility and Risk Register)  
**Parent Document:** [README.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/session_and_power_management/README.md)  
**Classification:** Core System Architecture / Operational Reliability Standard  

---

## 1. Lifecycle Compatibility Risk Register

Operating across diverse Windows power states, sleep modes, and user sessions introduces technical failure modes that must be addressed:

| Risk ID | Risk Summary & Failure Mode | Prob. | Impact | Severity | Architecture Mitigation & Prevention Strategy |
| :--- | :--- | :---: | :---: | :---: | :--- |
| **`RSK-LCE-01`** | **Wake Resumes Despite User Manual Pause**<br>User pauses wallpaper; system sleeps and resumes; wallpaper resumes unexpectedly, violating user intent. | High | Med | **HIGH** | **Composable Bitmask Architecture (`DEC-LCE-01`)**: `PauseReasonSet` separates `UserManual` from `SystemSleep`. Playback resumes only when bitmask is completely empty. |
| **`RSK-LCE-02`** | **DirectX Device Loss on System Wake**<br>GPU reinitializes during ACPI wake, invalidating D3D11 swapchains and causing player process crash. | Med | High | **CRITICAL** | **Staged Revalidation & Device Reset Handling (`DEC-LCE-02`)**: Player tests device status post-wake; recreates swapchain upon `DXGI_ERROR_DEVICE_RESET`. |
| **`RSK-LCE-03`** | **Sleep Preparation Timeout Violation**<br>Core blocks saving files during `PBT_APMSUSPEND`, causing Windows to force-suspend or crash application. | Med | High | **HIGH** | **Bounded $150\text{ ms}$ Sleep Budget (`DEC-LCE-03`)**: Asynchronous flushes execute with an unyielding 150 ms timeout before returning `TRUE` to Windows. |
| **`RSK-LCE-04`** | **High Background CPU on Display Sleep**<br>Display powers off but video decoders continue running at 60 FPS in background, draining laptop battery. | High | Med | **HIGH** | **`GUID_MONITOR_POWER_ON` Quiescent State (`DEC-LCE-04`)**: Display off signal pauses decoders and swapchains, dropping CPU overhead to $< 0.05\%$. |
| **`RSK-LCE-05`** | **Audio Popping / Desync on Workstation Lock**<br>Audio stream continues playing or loops glitchy audio buffer while computer is locked. | Med | Med | **MEDIUM** | **Immediate WASAPI Stop on Lock (`DEC-LCE-05`)**: `WTS_SESSION_LOCK` immediately attenuates volume and calls `IAudioClient::Stop()`. |
| **`RSK-LCE-06`** | **Stale WorkerW Handle Post-Sleep**<br>Explorer restarts during sleep; player resumes into dead HWND, rendering into an orphaned surface. | Med | High | **HIGH** | **Dual-Key Host Validation on Wake (`DEC-LCE-06`)**: Stage 1 of resume pipeline validates `WorkerW` via `IsWindow` and class check before unpausing. |
| **`RSK-LCE-07`** | **Sign-Out Deadlock Blocks Windows Logoff**<br>Engine attempts slow IPC shutdown during logoff, causing Windows to display "App is preventing logoff". | Med | High | **CRITICAL** | **Instant `WM_QUERYENDSESSION` Approval (`DEC-LCE-07`)**: Core returns `TRUE` in $< 5\text{ ms}$; completes shutdown via Job Object in $\le 200\text{ ms}$. |
| **`RSK-LCE-08`** | **Battery Threshold Flapping**<br>Battery level fluctuates around 20% under load, causing rapid pause/unpause toggles. | Med | Low | **MEDIUM** | **Hysteresis Threshold Gating (`DEC-LCE-08`)**: Pause triggers at $\le 20\%$; unpause requires charging to $\ge 25\%$. |

---

## 2. Architecture Decision Records (Lifecycle & Power: `DEC-LCE-01` to `08`)

The following formal decisions govern all Windows session and power management implementations across KinetiWall:

### `DEC-LCE-01`: Composable Pause Reason Bitmask Architecture
* **Context:** Multiple pause conditions (manual, battery, sleep, lock, fullscreen) can be active simultaneously.
* **Decision:** Model pause state as a `PauseReason` bitmask. Playback is permitted if and only if the bitmask is empty (`active_pause_reasons.is_empty()`).
* **Consequences:** Eliminates resume bugs; ensures explicit user manual pauses are never overridden by system resume or workstation unlock events.

### `DEC-LCE-02`: Staged Revalidation Pipeline Following System Resume
* **Context:** Hardware peripherals and graphics drivers initialize asynchronously upon waking from sleep.
* **Decision:** Implement a 5-stage revalidation pipeline (Host validation $\rightarrow$ Topology re-audit $\rightarrow$ Graphics device audit $\rightarrow$ Audio audit $\rightarrow$ Policy gate) before resuming playback.
* **Consequences:** Prevents device-loss crashes, black surfaces, and audio glitches following ACPI wake.

### `DEC-LCE-03`: Bounded $150\text{ ms}$ Sleep Preparation Deadline
* **Context:** Windows grants limited time for desktop applications to prepare for suspension during `PBT_APMSUSPEND`.
* **Decision:** Enforce an unyielding 150 ms internal budget for flushing configuration state and pausing players.
* **Consequences:** Ensures KinetiWall never blocks or delays Windows sleep or modern standby transitions.

### `DEC-LCE-04`: Passive Quiescent State for Display Power-Off
* **Context:** Running 60 FPS video wallpaper while monitors are asleep wastes power and generates heat.
* **Decision:** Intercept `GUID_MONITOR_POWER_ON` (0); halt swapchain presentation and media decoding while keeping player processes resident in memory.
* **Consequences:** Drops idle background CPU overhead to $< 0.05\%$, while enabling instant ($\le 50\text{ ms}$) resume upon mouse wake.

### `DEC-LCE-05`: Automatic WASAPI Audio Stream Handoff on Session Lock
* **Context:** Audio wallpaper playback during workstation lock wastes battery and compromises privacy.
* **Decision:** Upon `WTS_SESSION_LOCK`, immediately stop audio streams and release COM endpoint loopbacks.
* **Consequences:** Eliminates audio popping, avoids playing sound on locked machines, and preserves system resources.

### `DEC-LCE-06`: Dual-Key Window Handle Validation on System Resume
* **Context:** `explorer.exe` may restart during sleep, invalidating cached `WorkerW` host handles.
* **Decision:** Check `IsWindow(hWorkerW)` and `RealGetWindowClassW == "WorkerW"` before resuming presentation.
* **Consequences:** Detects dead host handles immediately; triggers automatic re-anchoring before frames are presented.

### `DEC-LCE-07`: Rapid Zero-Delay Logoff Teardown (`< 200 ms`)
* **Context:** Users expect instantaneous logoff and shutdown.
* **Decision:** Return `TRUE` to `WM_QUERYENDSESSION` within 5 ms; terminate child players via Job Object within 200 ms.
* **Consequences:** Complies with `KW-NF-006`; never triggers the "This app is preventing shutdown" Windows dialog.

### `DEC-LCE-08`: Battery Threshold Hysteresis Gating (20% Pause / 25% Resume)
* **Context:** Battery percentage fluctuations under load can cause rapid pause/unpause toggling.
* **Decision:** Implement a 5% hysteresis buffer: battery pause activates at $\le 20\%$; automatic resume requires charging to $\ge 25\%$.
* **Consequences:** Prevents power-state flapping and reduces wear on graphics decoders.

---

## 3. Open Engineering Review Items

| Item ID | Description | Current Architectural Status | Target Phase |
| :--- | :--- | :--- | :---: |
| **`REV-LCE-01`** | **Modern Standby (S0 Low Power Idle) Network Throttling:** Evaluate whether web wallpapers downloading remote assets need specific pause handling under S0 sleep. | Handled via `PBT_APMSUSPEND`; will be benchmarked on Modern Standby laptops. | Phase 4 (Runtime Verification) |
| **`REV-LCE-02`** | **Fast Startup (Hybrid Boot) Driver Recovery:** Validate whether Windows Fast Startup (hibernated kernel) requires full swapchain recreation upon boot. | Covered by Stage 3 device audit; scheduled for validation on clean Windows 11 installs. | Phase 4 (Runtime Verification) |
| **`REV-LCE-03`** | **Remote Desktop Protocol (RDP) Dynamic Bitrate:** Assess whether RDP session connections (`WTS_REMOTE_CONNECT`) should automatically throttle frame rate to 30 FPS. | Policy is specified; evaluating RDP network bandwidth telemetry. | Phase 4 (Runtime Verification) |

