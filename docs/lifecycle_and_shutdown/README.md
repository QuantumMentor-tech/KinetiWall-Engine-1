# KinetiWall — Lifecycle Management, Graceful Shutdown & Resource Cleanup Specification
**Document ID:** `KW-LFC-000`  
**Phase:** 3 (Observability, Diagnostics & Operational Reliability)  
**Step:** 15 of 90  
**Status:** Approved Architectural Specification  
**Classification:** Core System Architecture / Operational Reliability Standard  

---

## 1. Document Overview and Scope

This document suite establishes the authoritative engineering specification for process lifecycle states, startup orchestration, orderly shutdown sequences, deterministic resource reclamation, and orphan-process prevention in the **KinetiWall** desktop live wallpaper engine.

KinetiWall operates as a multi-process Windows subsystem with strict process boundaries:
1. **Core Controller Daemon (`kinetiwall-core.exe`):** Unprivileged background supervisor, notification-area tray anchor, IPC coordinator, single-writer storage manager, and hardware event monitor.
2. **Ephemeral Configuration and Gallery UI (`kinetiwall-ui.exe`):** On-demand GUI application providing library management, settings adjustment, and performance dashboards. Shuts down cleanly on window closure with zero persistent resident memory.
3. **Dedicated Per-Display Wallpaper Players (`kinetiwall-player-video.exe`, `kinetiwall-player-web.exe`):** Isolated rendering and decoding processes spawned per active display, rendering directly into desktop `WorkerW` / `Progman` hierarchy.

Because KinetiWall integrates deeply with Windows shell windows, DirectX swapchains, media hardware decoders, and OS session broadcasts, lifecycle anomalies (such as orphan renderers running invisibly behind desktop icons, frozen WorkerW surfaces, deadlocked IPC sockets, or zombie processes holding SQLite file locks) represent catastrophic failure modes. This specification formalizes deterministic lifecycle mechanics and zero-orphan guarantees.

---

## 2. Core Lifecycle Principles (Principles A through L)

The lifecycle architecture of KinetiWall is governed by twelve inviolable engineering principles:

* **Principle A: Single Authoritative Supervisor**  
  `kinetiwall-core.exe` is the sole lifecycle supervisor for all wallpaper player instances, background renderers, and transient helper workers. No worker or player process may spawn sibling players or self-replicate.
* **Principle B: Ephemeral UI Separation**  
  Closing the settings or library window terminates `kinetiwall-ui.exe` completely, reclaiming 100% of its working set. UI termination never interrupts or degrades active wallpaper playback.
* **Principle C: Deterministic Windows Job Object Supervision**  
  Every child process spawned by `kinetiwall-core.exe` is irrevocably bound upon creation to a Windows Job Object configured with `JOB_OBJECT_LIMIT_KILL_ON_JOB_CLOSE`. In the event of catastrophic Core process termination, power loss, or external termination, the Windows NT kernel terminates all child processes in under 50 ms.
* **Principle D: Multi-Phase Bounded Shutdown with Escalation**  
  Shutdown proceeds strictly through ordered phases (Notification $\rightarrow$ Cooperative Pause $\rightarrow$ Flush $\rightarrow$ Disconnect $\rightarrow$ Exit). Every cooperative step has an unyielding timeout ($T \le 250\text{ ms}$). If a subprocess fails to exit cooperatively within budget, the supervisor escalates deterministically to `TerminateProcess`.
* **Principle E: Shell and Desktop Restoration Guarantee**  
  Prior to player process teardown, wallpaper surfaces must be detached from `WorkerW` surfaces or hidden cleanly, restoring the user's native static Windows wallpaper without screen flicker, shell restart, or `explorer.exe` interference.
* **Principle F: Process-Instance Identity Immunity to PID Recycling**  
  All process monitoring, supervision tables, and IPC sessions reference globally unique, cryptographically random 128-bit Process Instance UUIDs (`ProcessInstanceId`) rather than raw OS PIDs, preventing race conditions caused by Windows OS PID recycling.
* **Principle G: Atomic State Flushes Prior to Exit**  
  In-memory configuration modifications, display state changes, and diagnostics metrics must be persisted via atomic replace (`MoveFileExW` / `REPLACEFILE_WRITE_THROUGH`) before IPC teardown. If a write fails due to filesystem locks, the failure is explicitly logged; no false success is returned.
* **Principle H: Session and Power Awareness**  
  The engine intercepts Windows session notifications (`WM_QUERYENDSESSION`, `WM_ENDSESSION`), power state transitions (`GUID_MONITOR_POWER_ON`, `PBT_APMSUSPEND`, `PBT_APMRESUMEAUTOMATIC`), and display topology changes (`WM_DISPLAYCHANGE`), freezing rendering engines during sleep/display-off and entering quiescent states without thrashing or leaking GPU resources.
* **Principle I: Deterministic Resource Reclamation Hierarchy**  
  Every allocated system resource (DirectX swapchains, mpv instances, WebView2 environments, audio client loopbacks, named pipe handles, WinEvent hooks, and memory allocations) possesses an explicit ownership contract, destruction priority order, and leak detection mechanism.
* **Principle J: Crash and Restart Flood Prevention**  
  Subprocesses that terminate unexpectedly or crash are subject to an exponential-backoff and circuit-breaker restart policy (maximum 3 restarts in 60 seconds). A crashing player transitions the affected display to a quiescent fallback state rather than triggering restart loops.
* **Principle K: Non-Destructive Startup Reconciliation**  
  Upon cold startup, `kinetiwall-core.exe` scans the OS process table for orphaned KinetiWall instances from prior abnormal crashes. Stale instances are verified using executable image path hashes and matching command-line session signatures before termination, ensuring unrelated third-party processes are never touched.
* **Principle L: Auditable Lifecycle Telemetry**  
  Every lifecycle transition (State Changes, Shutdown Milestones, Escalations, Timeout Violations, Resource Leaks) emits structured diagnostics events with monotonic millisecond timestamps, Process UUIDs, and Windows error codes.

---

## 3. Documentation Suite Index

| Deliverable | File Path | Focus & Architectural Scope |
| :--- | :--- | :--- |
| **Index & Principles** | [README.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/lifecycle_and_shutdown/README.md) | Architectural index, 12 core lifecycle principles, requirements matrix. |
| **Deliverable A** | [PROCESS_LIFECYCLE_SPECIFICATION.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/lifecycle_and_shutdown/PROCESS_LIFECYCLE_SPECIFICATION.md) | Component inventory, 14 lifecycle states, startup orchestration, UI closure, shutdown scopes, timeouts, OS lifecycle events. |
| **Deliverable B** | [SHUTDOWN_AND_CLEANUP_PROTOCOL.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/lifecycle_and_shutdown/SHUTDOWN_AND_CLEANUP_PROTOCOL.md) | 15 ordered teardown stages, component shutdown responsibilities, cancellation behavior, atomic persistence guarantees, escalation mechanics. |
| **Deliverable C** | [RESOURCE_OWNERSHIP_MATRIX.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/lifecycle_and_shutdown/RESOURCE_OWNERSHIP_MATRIX.md) | Resource inventory (Process, IPC, Win32 HWND, D3D11, mpv, WebView2, WASAPI, Files), ownership lifetimes, release order, timeouts, leak detection. |
| **Deliverable D** | [ORPHAN_PROCESS_PREVENTION.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/lifecycle_and_shutdown/ORPHAN_PROCESS_PREVENTION.md) | Windows Job Object encapsulation, PID recycling mitigation via UUIDs, async process exit wait loops, cold startup reconciliation, safety verification. |
| **Deliverable E** | [LIFECYCLE_CLEANUP_TEST_PLAN.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/lifecycle_and_shutdown/LIFECYCLE_CLEANUP_TEST_PLAN.md) | Verification methodology, acceptance criteria matrix (`LIFE-001` to `LIFE-012`), unit/integration/E2E test cases, fault injection scenarios. |
| **Deliverable F** | [LIFECYCLE_RISK_AND_DECISION_LOG.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/lifecycle_and_shutdown/LIFECYCLE_RISK_AND_DECISION_LOG.md) | Failure modes, deadlock risks, lost-write risks, decision records (`DEC-LFC-01` to `DEC-LFC-08`), open engineering items. |

---

## 4. Requirements Traceability Matrix

The specifications defined in this document suite directly satisfy and enforce the following system requirements and architecture decisions:

| Requirement / Decision | Description | Governing Specification Section |
| :--- | :--- | :--- |
| **`KW-F-001`** | Core engine initialization and native desktop shell integration | `PROCESS_LIFECYCLE_SPECIFICATION.md` §3 |
| **`KW-F-002`** | High-performance video wallpaper playback engine (mpv / D3D11) | `RESOURCE_OWNERSHIP_MATRIX.md` §3 |
| **`KW-F-003`** | Interactive web wallpaper playback engine (WebView2) | `RESOURCE_OWNERSHIP_MATRIX.md` §4 |
| **`KW-F-007`** | Multi-monitor independent display configuration & lifecycle | `PROCESS_LIFECYCLE_SPECIFICATION.md` §7 |
| **`KW-F-008`** | Power-state and fullscreen hardware occlusion detection | `PROCESS_LIFECYCLE_SPECIFICATION.md` §8 |
| **`KW-NF-001`** | CPU utilization limits ($\le 1.0\%$ idle background) | `ORPHAN_PROCESS_PREVENTION.md` §3 |
| **`KW-NF-002`** | Memory footprint targets ($\le 60\text{ MB}$ Core, $0\text{ MB}$ closed UI) | `PROCESS_LIFECYCLE_SPECIFICATION.md` §4 |
| **`KW-NF-006`** | Graceful shutdown and rapid exit response ($\le 250\text{ ms}$) | `SHUTDOWN_AND_CLEANUP_PROTOCOL.md` §6 |
| **`KW-NG-001`** | UI ephemeral memory guarantee ($0\text{ MB}$ resident footprint) | `PROCESS_LIFECYCLE_SPECIFICATION.md` §4 |
| **`ADR-0002`** | Isolated multi-process topology (Core, UI, Per-Display Players) | `PROCESS_LIFECYCLE_SPECIFICATION.md` §2 |
| **`DEC-PRF-08`** | Windows Job Object orphan process containment | `ORPHAN_PROCESS_PREVENTION.md` §2 |
| **`DEC-RCV-05`** | Bounded shutdown timeout and unyielding force-termination escalation | `SHUTDOWN_AND_CLEANUP_PROTOCOL.md` §6 |
| **`DEC-LFC-01..08`** | Lifecycle decisions covering UUIDs, atomic flushes, Job Objects | `LIFECYCLE_RISK_AND_DECISION_LOG.md` §3 |

