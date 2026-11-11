# Windows Session, Power Events, Sleep/Resume & Lifecycle Integration Specification
**Document ID:** `KW-LCE-000`  
**Phase:** 4 (Windows Platform Integration & Desktop Hosting)  
**Step:** 20 of 90  
**Status:** Approved Architectural Specification  
**Classification:** Windows Session Management, Power Policy & Lifecycle Integration Standard  

---

## 1. Document Overview and Scope

This document suite establishes the authoritative engineering specification for **Windows Session Lifecycles, Power-Management Broadcasts, System Sleep & Resume Transitions, Display Sleep & Wake Cycles, Workstation Lock & Unlock Coordination, Battery-Aware Degradation, and Pause-Policy Precedence** for the **KinetiWall** live wallpaper engine.

As an unprivileged, multi-process desktop subsystem running continuously in the user session, KinetiWall must interact harmoniously with the Windows operating system across diverse hardware power states:
1. **System Power Transitions:** Handling OS suspend/sleep (`PBT_APMSUSPEND`), resume automatic (`PBT_APMRESUMEAUTOMATIC`), resume suspend (`PBT_APMRESUMESUSPEND`), AC-to-battery switches, and low-battery thresholds.
2. **Display Power Transitions:** Differentiating full system sleep from display-only power-off (`GUID_MONITOR_POWER_ON` = 0) and display wake (`GUID_MONITOR_POWER_ON` = 1).
3. **Session State Transitions:** Handling workstation lock (`WTS_SESSION_LOCK`), unlock (`WTS_SESSION_UNLOCK`), user sign-out (`WM_QUERYENDSESSION` / `WM_ENDSESSION`), fast user switching, and remote desktop disconnections.
4. **Policy Precedence & Pause Stack:** Establishing a strict, non-overriding precedence hierarchy between explicit user intent (Manual Stop / Pause), battery conservation, fullscreen occlusion, and OS power transitions.
5. **Deterministic Hardware Resource Revalidation:** Detecting invalidated Direct3D 11 swapchains, reset media decoders (mpv), halted WASAPI audio endpoints, and stale `WorkerW` host handles following system resume.

This specification formalizes the event ownership model, defines an 8-state canonical lifecycle state machine, establishes a deterministic 15-event policy matrix, specifies revalidation pipelines, and institutes a 15-point verification matrix.

---

## 2. Core Windows Lifecycle Principles (Principles A through L)

The session and power integration architecture of KinetiWall is strictly governed by twelve engineering principles:

* **Principle A: Unconditional Operating System Cooperation**  
  KinetiWall must never attempt to block, delay, or veto Windows sleep, shutdown, or logoff. Calls to `WM_QUERYENDSESSION` return `TRUE` immediately ($< 5\text{ ms}$), facilitating rapid OS shutdown within the $250\text{ ms}$ budget (`KW-NF-006`).
* **Principle B: Inviolability of Explicit User Intent**  
  A wallpaper stopped or paused explicitly by the user must **never** automatically resume simply because the system wakes from sleep, the display powers on, or the user unlocks the workstation. User intent is permanent until explicitly cleared.
* **Principle C: Process Independence & Ephemeral UI Isolation**  
  Lifecycle events affecting the configuration GUI (`kinetiwall-ui.exe`) must never destabilize or terminate active wallpaper players. Closing the UI window terminates the UI process completely ($0.0\text{ MB}$ footprint) while `kinetiwall-core.exe` maintains authoritative power and session supervision.
* **Principle D: Clear Demarcation of Distinct Lifecycle Conditions**  
  System sleep, display sleep, workstation lock, user sign-out, and fullscreen gaming occlusion are structurally distinct operating states. The engine must never treat a display power-off as a full system sleep or a workstation lock as a sign-out.
* **Principle E: Authoritative Revalidation Following Resume**  
  Receipt of an OS resume notification (`PBT_APMRESUMEAUTOMATIC`) is **not** proof that graphics hardware, display topology, or audio devices are operational. The engine executes a staged revalidation pipeline before resuming playback.
* **Principle F: Anti-Thrashing Bounded Transition Work**  
  Event-driven transition routines, device queries, and player re-anchoring operations must be bounded by unyielding timeouts ($\le 250\text{ ms}$) and protected by anti-oscillation debounce timers, preventing restart storms.
* **Principle G: Passive Event-Driven Monitoring (Zero Idle Wake-Ups)**  
  Power and session monitoring rely exclusively on passive Win32 event subscriptions (`RegisterPowerSettingNotification`, `WTSRegisterSessionNotification`, `WM_POWERBROADCAST`), guaranteeing $\le 0.05\%$ idle CPU overhead (`KW-NF-001`).
* **Principle H: Desktop Shell Safety Invariant**  
  If graphics hardware or the desktop `WorkerW` host fails to recover post-resume, KinetiWall drops into a safe degraded state, cleanly exposing the native static Windows wallpaper rather than leaving blank, black, or floating windows.
* **Principle I: Observable & Redacted Diagnostics**  
  Every session and power transition emits structured telemetry recording the event source, power state, and recovery duration. Sensitive personal information (usernames, session credentials, private file paths) is strictly redacted.
* **Principle J: Atomic Data Protection Prior to Sleep/Shutdown**  
  Pending configuration changes and playlist modifications must be flushed atomically to disk (`MoveFileExW`) upon receipt of sleep or shutdown notifications, preventing corrupted or truncated files.
* **Principle K: Multi-Reason Composable Pause Stack**  
  Pause states are modeled as a composable bitmask set (`PauseReason`). A wallpaper only transitions to active rendering when all active pause reasons (`UserManual`, `BatterySaving`, `FullscreenOccluded`, `DisplayPowerOff`, `SystemSleep`, `SessionLocked`) have been cleared.
* **Principle L: Hardware-Specific Lifecycle Adaptation**  
  DirectX 11 swapchains, mpv video decoders, and WebView2 instances are managed according to their documented hardware failure modes, handling `DXGI_ERROR_DEVICE_RESET` gracefully without crashing supervisor daemons.

---

## 3. Documentation Suite Index

| Deliverable | File Path | Focus & Architectural Scope |
| :--- | :--- | :--- |
| **Index & Principles** | [README.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/session_and_power_management/README.md) | Architectural index, 12 core lifecycle principles, requirements traceability matrix. |
| **Deliverable A** | [WINDOWS_SESSION_POWER_INTEGRATION.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/session_and_power_management/WINDOWS_SESSION_POWER_INTEGRATION.md) | Native notification mechanisms, subscription ownership in Core, canonical 8-state lifecycle model, and IPC dispatch. |
| **Deliverable B** | [SLEEP_RESUME_RECOVERY_POLICY.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/session_and_power_management/SLEEP_RESUME_RECOVERY_POLICY.md) | Staged revalidation pipeline post-resume, graphics device loss recovery (`DXGI_ERROR_DEVICE_RESET`), audio loopback recovery, and degraded fallback states. |
| **Deliverable C** | [SESSION_POWER_POLICY_MATRIX.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/session_and_power_management/SESSION_POWER_POLICY_MATRIX.md) | Authoritative 15-event matrix mapping OS notifications to exact player, renderer, audio, and window behaviors. |
| **Deliverable D** | [LIFECYCLE_POLICY_PRECEDENCE.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/session_and_power_management/LIFECYCLE_POLICY_PRECEDENCE.md) | Multi-reason composable pause stack, battery-saving policy thresholds, fullscreen interaction, and user-intent preservation rules. |
| **Deliverable E** | [WINDOWS_POWER_SESSION_TEST_PLAN.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/session_and_power_management/WINDOWS_POWER_SESSION_TEST_PLAN.md) | 15-point verification matrix (`PWR-001` through `PWR-015`), fault-injection scenarios, synthetic notification injection, and CI automation. |
| **Deliverable F** | [WINDOWS_LIFECYCLE_COMPATIBILITY_RISKS.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/session_and_power_management/WINDOWS_LIFECYCLE_COMPATIBILITY_RISKS.md) | Risk register (`RSK-LCE-01..08`), Architecture Decision Records (`DEC-LCE-01..08`), and open engineering review items. |

---

## 4. Requirements Traceability Matrix

| Requirement / Decision | Description | Governing Specification Section |
| :--- | :--- | :--- |
| **`KW-F-001`** | Core engine initialization and native desktop shell integration | `WINDOWS_SESSION_POWER_INTEGRATION.md` §2 |
| **`KW-F-008`** | Power-state and fullscreen hardware occlusion detection | `SESSION_POWER_POLICY_MATRIX.md` §2 |
| **`KW-NF-001`** | CPU utilization limits ($\le 1.0\%$ idle background overhead) | `WINDOWS_SESSION_POWER_INTEGRATION.md` §3 |
| **`KW-NF-006`** | Graceful shutdown and rapid exit response ($\le 250\text{ ms}$) | `SESSION_POWER_POLICY_MATRIX.md` §2 |
| **`KW-NG-001`** | UI ephemeral memory guarantee ($0\text{ MB}$ resident footprint) | `WINDOWS_SESSION_POWER_INTEGRATION.md` §2 |
| **`ADR-0002`** | Isolated multi-process topology (Core, UI, Per-Display Players) | `WINDOWS_SESSION_POWER_INTEGRATION.md` §2 |
| **`DEC-RCV-05`** | Bounded shutdown timeout and force-termination escalation | `SESSION_POWER_POLICY_MATRIX.md` §2 |
| **`DEC-LFC-06`** | Quiescent state entry on session lock and monitor standby | `SLEEP_RESUME_RECOVERY_POLICY.md` §3 |
| **`DEC-DSP-02`** | Monotonic 500ms anti-oscillation debounce for display changes | `SLEEP_RESUME_RECOVERY_POLICY.md` §2 |
| **`DEC-LCE-01..08`**| Dedicated power management, sleep recovery, and pause precedence decisions | `WINDOWS_LIFECYCLE_COMPATIBILITY_RISKS.md` §2 |

