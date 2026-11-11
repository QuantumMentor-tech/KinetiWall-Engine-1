# Windows Power & Session Lifecycle Test Plan
**Document ID:** `KW-LCE-005`  
**Deliverable:** E (Power and Session Lifecycle Test Plan)  
**Parent Document:** [README.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/session_and_power_management/README.md)  
**Classification:** Quality Assurance & System Verification Specification  

---

## 1. Test Strategy & Acceptance Framework

Testing power management, sleep, and session lifecycles requires specialized verification techniques to simulate OS broadcasts without disrupting physical test machines:
1. **Tier 1: Pure Bitmask Unit Tests** (Validates `PauseReason` bitwise logic, precedence resolution, and mask transitions).
2. **Tier 2: Synthetic Message Injection** (Dispatches synthetic `WM_POWERBROADCAST` and `WM_WTSSESSION_CHANGE` messages to Core's hidden window).
3. **Tier 3: Windows Virtual Machine Power Transitions** (Executes controlled ACPI S3 sleep and wake cycles inside Windows Hyper-V/VirtualBox test VMs).
4. **Tier 4: Adversarial Power Chaos** (Injects sleep signals during active wallpaper switching, rapid lock/unlock bursts, and device reset faults).

> [!IMPORTANT]
> All test cases below represent approved verification specifications. Their status is marked as `NOT RUN (DESIGN PHASE)` pending runtime implementation in Phase 4.

---

## 2. Acceptance Criteria Verification Matrix (`PWR-001` through `PWR-015`)

| Test ID | Test Scenario | Primary Target | Observable Acceptance Criteria | Status |
| :--- | :--- | :--- | :--- | :--- |
| **`PWR-001`** | **Pre-Sleep Bounded Flush** | `PBT_APMSUSPEND` | Core commits `config.json` via `MoveFileExW` and pauses all players within $\le 150\text{ ms}$; returns `TRUE` to OS. | `NOT RUN (DESIGN PHASE)` |
| **`PWR-002`** | **Post-Resume Staged Revalidation** | `PBT_APMRESUME` | System resume revalidates `WorkerW` HWND and D3D11 device; resumes 60 FPS playback within $\le 250\text{ ms}$. | `NOT RUN (DESIGN PHASE)` |
| **`PWR-003`** | **Display Sleep Quiescence** | `GUID_MONITOR_POWER_ON` (0)| Powering off monitors halts swapchain `Present`; background CPU usage drops to $< 0.05\%$ within $\le 20\text{ ms}$. | `NOT RUN (DESIGN PHASE)` |
| **`PWR-004`** | **Display Wake Instant Resume** | `GUID_MONITOR_POWER_ON` (1)| Moving mouse to wake display resumes swapchain rendering within $\le 50\text{ ms}$ without recreating player processes. | `NOT RUN (DESIGN PHASE)` |
| **`PWR-005`** | **Workstation Lock Muting** | `WTS_SESSION_LOCK` | Locking desktop (`Win+L`) freezes video decoders and halts WASAPI audio loopback streams within $\le 20\text{ ms}$. | `NOT RUN (DESIGN PHASE)` |
| **`PWR-006`** | **Workstation Unlock Restoration** | `WTS_SESSION_UNLOCK` | Unlocking workstation unfreezes rendering immediately if no other pause reasons are active. | `NOT RUN (DESIGN PHASE)` |
| **`PWR-007`** | **Manual User Pause Invariance** | User Intent Protection | User clicks "Pause"; system enters sleep and wakes; wallpaper **remains paused** because `UserManual` bit is preserved. | `NOT RUN (DESIGN PHASE)` |
| **`PWR-008`** | **Battery-Aware Throttle / Pause** | `GUID_ACDC_POWER_SOURCE` | Unplugging AC laptop cord pauses video wallpaper (or throttles to 30 FPS) according to user power settings. | `NOT RUN (DESIGN PHASE)` |
| **`PWR-009`** | **Critical Low-Battery Fallback** | `GUID_BATTERY_PERCENTAGE` | Battery level dropping below 15% pauses wallpaper and restores native static Windows wallpaper to conserve battery. | `NOT RUN (DESIGN PHASE)` |
| **`PWR-010`** | **D3D11 Device Reset Recovery** | `DXGI_ERROR_DEVICE_RESET` | Injecting graphics device reset on resume causes player to reinitialize D3D11 swapchains cleanly without crashing Core. | `NOT RUN (DESIGN PHASE)` |
| **`PWR-011`** | **Explorer Restart Post-Sleep** | Explorer Shell Hook | Explorer crashing during sleep is detected on wake via `TaskbarCreated`; Core re-anchors to new `WorkerW` in $\le 500\text{ ms}$. | `NOT RUN (DESIGN PHASE)` |
| **`PWR-012`** | **User Sign-Out Fast Teardown** | `WM_QUERYENDSESSION` | User logoff returns `TRUE` in $< 5\text{ ms}$; completes clean process termination in $\le 200\text{ ms}$ without blocking logoff. | `NOT RUN (DESIGN PHASE)` |
| **`PWR-013`** | **Fast User Switching Isolation** | `WTS_CONSOLE_DISCONNECT` | Switching to another Windows user pauses renderers; switching back (`WTS_CONSOLE_CONNECT`) restores playback cleanly. | `NOT RUN (DESIGN PHASE)` |
| **`PWR-014`** | **Rapid Lock/Unlock Burst Immunity**| Anti-Thrashing Guard | Injecting 5 rapid lock/unlock events in 1 second does not crash players, leak handles, or drop IPC connections. | `NOT RUN (DESIGN PHASE)` |
| **`PWR-015`** | **Zero Battery Telemetry Leakage** | Privacy Protection | Telemetry logs record power state transitions without logging sensitive user credentials, names, or file paths. | `NOT RUN (DESIGN PHASE)` |

---

## 3. Synthetic Test Harness Procedures

### 3.1 Test `PWR-007`: User Intent Preservation Under System Sleep

* **Objective:** Prove that `UserManual` pause reason survives a system sleep and resume cycle.
* **Procedure:**
  1. Set active video wallpaper. Verify 60 FPS rendering (`STATE_RUNNING`).
  2. Send IPC command `CMD_USER_PAUSE`. Assert `active_pause_reasons.contains(UserManual)`.
  3. Synthesize message `WM_POWERBROADCAST` (`PBT_APMSUSPEND`). Assert `SystemSleep` bit added.
  4. Synthesize message `WM_POWERBROADCAST` (`PBT_APMRESUMEAUTOMATIC`). Assert `SystemSleep` bit removed.
  5. Check `is_playback_permitted()`.
* **Pass Condition:** `is_playback_permitted() == false`. Player remains in quiescent paused state. Wallpaper does not resume until user sends `CMD_USER_RESUME`.

### 3.2 Test `PWR-010`: Direct3D 11 Device Reset Chaos Test

* **Objective:** Verify graceful recovery when GPU driver reinitializes during system wake.
* **Procedure:**
  1. Trigger test mock `simulate_dxgi_device_loss()`.
  2. Next swapchain `Present` call returns `DXGI_ERROR_DEVICE_RESET`.
  3. Player detects error, destroys old swapchain, re-queries `IDXGIFactory`, and recreates swapchain.
* **Pass Condition:** Player recovers rendering within $200\text{ ms}$; Core logs `WARN_D3D11_DEVICE_RESET_RECOVERED`; zero unhandled panics.

