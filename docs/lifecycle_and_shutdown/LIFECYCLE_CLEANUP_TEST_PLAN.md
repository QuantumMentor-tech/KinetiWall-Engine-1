# Lifecycle & Cleanup Validation Plan & Test Matrix
**Document ID:** `KW-LFC-005`  
**Deliverable:** E (Lifecycle Cleanup Test Plan)  
**Parent Document:** [README.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/lifecycle_and_shutdown/README.md)  
**Classification:** Quality Assurance & System Verification Specification  

---

## 1. Test Strategy & Acceptance Framework

This test plan defines the formal verification matrix for validating process lifecycles, shutdown sequences, resource reclamation, and orphan process prevention in KinetiWall.

Testing is structured across four rigorous tiers:
1. **Tier 1: Unit & Component Tests** (Mocked Win32 handles, IPC message parsers, state machine transitions).
2. **Tier 2: Subsystem Integration Tests** (DirectX swapchain destruction, mpv shutdown hooks, WebView2 navigation resets).
3. **Tier 3: End-to-End System Scenarios** (Multi-monitor startup/teardown, UI close vs engine exit, session lock/unlock).
4. **Tier 4: Adversarial Fault Injection** (Core hard crash `kill -9`, timeout simulated deadlocks, explorer.exe restarts).

> [!IMPORTANT]
> In accordance with the Project Phase 1–3 charter, all test cases below represent approved verification specifications. Their status is marked as `NOT RUN (DESIGN PHASE)` pending runtime implementation in Phase 4.

---

## 2. Acceptance Criteria Verification Matrix (`LIFE-001` through `LIFE-012`)

| Test ID | Test Title | Primary Target & Scope | Expected Acceptance Criteria | Status |
| :--- | :--- | :--- | :--- | :--- |
| **`LIFE-001`** | **Ephemeral UI Memory Reclamation** | `kinetiwall-ui.exe` | UI window closure results in 100% process termination within $150\text{ ms}$; resident memory drops to exactly $0.0\text{ MB}$; active wallpaper playback is uninterrupted. | `NOT RUN (DESIGN PHASE)` |
| **`LIFE-002`** | **Graceful Shutdown Duration Budget** | `kinetiwall-core.exe` & Players | Complete engine shutdown initiated via tray menu completes all 15 stages within $\le 250\text{ ms}$ (hard limit $500\text{ ms}$). | `NOT RUN (DESIGN PHASE)` |
| **`LIFE-003`** | **Windows Job Object Orphan Containment**| Windows Job Object | Terminating `kinetiwall-core.exe` forcefully (`taskkill /F /IM kinetiwall-core.exe`) results in the Windows NT kernel terminating all child players within $\le 50\text{ ms}$. Zero zombie renderers. | `NOT RUN (DESIGN PHASE)` |
| **`LIFE-004`** | **Shell Desktop Surface Restoration** | `WorkerW` / Shell Desktop | Following wallpaper stop or player exit, the `WorkerW` window is hidden/detached, instantly revealing the native Windows wallpaper without screen flicker or shell restart. | `NOT RUN (DESIGN PHASE)` |
| **`LIFE-005`** | **Zero Win32 Handle Leakage** | Core & Player Processes | Repeated startup and shutdown cycles (100 iterations) exhibit zero net change in GDI handles, USER handles, or kernel object handles (`Delta = 0`). | `NOT RUN (DESIGN PHASE)` |
| **`LIFE-006`** | **DirectX Swapchain Teardown** | D3D11 / DXGI Subsystem | Upon player stop, `ReportLiveDeviceObjects()` reports 0 active textures, render targets, or swapchains. VRAM allocation drops to $0\text{ MB}$. | `NOT RUN (DESIGN PHASE)` |
| **`LIFE-007`** | **Subprocess Escalation Protocol** | Core Supervisor Escalation | A hung player process (simulated infinite loop) is forcefully terminated via `TerminateProcess` after exactly $250\text{ ms}$ timeout. Error logged in telemetry. | `NOT RUN (DESIGN PHASE)` |
| **`LIFE-008`** | **Atomic Persistence on Shutdown** | Storage Engine (`config.json`)| In-memory settings modified right before exit are safely written to disk via atomic replace (`MoveFileExW`). Zero corrupted or empty config files. | `NOT RUN (DESIGN PHASE)` |
| **`LIFE-009`** | **Session Lock Quiescent State** | Power & Session Monitor | Triggering workstation lock (`WTS_SESSION_LOCK`) pauses all rendering loops, drops player CPU usage to $< 0.1\%$, and frees audio endpoints. Resumes cleanly on unlock. | `NOT RUN (DESIGN PHASE)` |
| **`LIFE-010`** | **Cold Boot Orphan Reconciliation** | Core Startup Scanner | Introducing a mock orphan `kinetiwall-player-video.exe` into the process tree prior to Core boot results in Core discovering and safely terminating the orphan during Stage 3. | `NOT RUN (DESIGN PHASE)` |
| **`LIFE-011`** | **Shell Restart Resilience** | Shell Hook (`TaskbarCreated`) | Killing and restarting `explorer.exe` causes Core to re-detect shell handles, recreate the tray notification icon, and re-anchor players without interrupting video playback. | `NOT RUN (DESIGN PHASE)` |
| **`LIFE-012`** | **PID Recycling Immunity** | Process Supervisor | Core rejects process commands or terminations if target process start time does not match recorded `creation_time`, preventing accidental kills of recycled PIDs. | `NOT RUN (DESIGN PHASE)` |

---

## 3. Test Scenarios and Execution Procedures

### 3.1 Test Scenario `LIFE-001`: Ephemeral UI Teardown & Isolation

* **Objective:** Verify that closing the UI window completely terminates `kinetiwall-ui.exe` and reclaims all memory without disturbing active wallpapers.
* **Preconditions:** Core is running; 2 displays actively rendering video wallpapers at 60 FPS; UI is opened by user.
* **Procedure:**
  1. Record baseline Core PID, Player PIDs, and system RAM usage.
  2. Dispatch `WM_CLOSE` to `kinetiwall-ui.exe` main window.
  3. Poll Windows process table for `kinetiwall-ui.exe`.
  4. Measure timestamp until process handle is signaled.
  5. Check frame render timers on Player processes.
* **Pass Criteria:**
  - `kinetiwall-ui.exe` exits within $150\text{ ms}$.
  - Memory consumption of UI drops to $0.0\text{ MB}$ resident.
  - Video player frame rates on Display 1 and 2 experience zero dropped frames ($\Delta \text{FPS} = 0$).

### 3.2 Test Scenario `LIFE-003`: Hard Crash Orphan Containment (Adversarial)

* **Objective:** Prove that unexpected termination of the supervisor daemon results in kernel-enforced cleanup of all child players via Windows Job Objects.
* **Preconditions:** Core running with 3 child player processes (2 video, 1 web).
* **Procedure:**
  1. Record PIDs of Core and all 3 player processes.
  2. Execute hard ungraceful termination of Core: `taskkill /F /PID <CorePID>`.
  3. Immediately begin polling the OS process table for the 3 player PIDs at $10\text{ ms}$ intervals.
* **Pass Criteria:**
  - Core terminates immediately.
  - All 3 player PIDs disappear from the OS process table within $\le 50\text{ ms}$ of Core's death.
  - No zombie `kinetiwall-player-*.exe` processes remain active.

### 3.3 Test Scenario `LIFE-007`: Hung Subprocess Shutdown Escalation

* **Objective:** Verify supervisor timeout enforcement and deterministic escalation to `TerminateProcess`.
* **Preconditions:** Test player subprocess configured with debug flag `--simulate-shutdown-hang`.
* **Procedure:**
  1. Core issues `CMD_TERMINATE_PLAYER`.
  2. The hung player receives the command but deliberately enters an infinite sleep loop without exiting.
  3. Monitor Core supervision timer.
* **Pass Criteria:**
  - Core awaits cooperative exit for exactly $250\text{ ms}$.
  - At $T = 250\text{ ms}$, Core logs `SHUTDOWN_TIMEOUT_ESCALATION`.
  - Core invokes `TerminateProcess(hPlayer, 0xFE)`.
  - Player process terminates within $20\text{ ms}$ of escalation.
  - Core cleans up process registry and exits or marks slot idle.

---

## 4. Test Harness & Automation Architecture

The lifecycle test suite is automated using a specialized Rust integration harness (`tests/lifecycle_harness/`):

```
+--------------------------------------------------------------------------+
|                    Rust Test Harness: `cargo test --test lifecycle`       |
+--------------------------------------------------------------------------+
|  1. Win32 Environment Setup:                                             |
|     - Mock Virtual Displays (using Windows Indirect Display Driver)      |
|     - Dedicated Job Object Sandbox                                       |
|  2. Process Observer Thread:                                             |
|     - Real-time sampling of PIDs, Handles, VRAM, and GDI/USER objects    |
|  3. Chaos Injector:                                                      |
|     - Simulated pipe disconnections                                      |
|     - Artificial 500ms deadlocks in COM/DirectX calls                    |
|     - Kernel process termination signals (TerminateProcess)              |
|  4. Invariant Evaluator:                                                 |
|     - Verifies LIFE-001 through LIFE-012 criteria                        |
|     - Generates Markdown verification report                             |
+--------------------------------------------------------------------------+
```

