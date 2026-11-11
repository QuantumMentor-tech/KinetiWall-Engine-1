# Dashboard State Integration, Live Status Synchronization & Reliability Specification
**Document ID:** `KW-DSI-000`  
**Phase:** 4 (UI/UX Design System & Dashboard)  
**Step:** 24 of 90  
**Status:** Approved Architectural Specification  
**Classification:** Frontend Application Architecture / State Management & Synchronization Standard  

---

## 1. Document Overview and Scope

This document suite establishes the authoritative engineering specification for **Dashboard State Integration, Live Status Synchronization & Reliability** (`KW-DSI-xxx`) within the KinetiWall desktop live wallpaper architecture.

Building upon Step 23's UI/UX presentation baseline ([DASHBOARD_ARCHITECTURE_AND_LAYOUT.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/dashboard_experience/DASHBOARD_ARCHITECTURE_AND_LAYOUT.md)), this specification details the rigorous bidirectional communication, state synchronization contracts, action lifecycle state machines, and fault recovery mechanisms that bind the presentation dashboard (`kinetiwall-ui.exe`) to the native core background engine (`kinetiwall-core.exe`).

The primary architectural challenge addressed in this step is maintaining absolute state consistency across isolated process boundaries without introducing background CPU polling, without permitting stale out-of-order IPC responses to overwrite modern state, and without allowing ephemeral UI window lifecycles to jeopardize active wallpaper playback.

---

## 2. Core State Integration Principles (Principles A through L)

The state integration and synchronization architecture is governed by twelve immutable engineering principles:

* **Principle A: Single Source of Truth (Core Authoritative State)**  
  `kinetiwall-core.exe` is the sole authoritative owner of all operational, playback, and configuration state. The frontend UI store (`Zustand`) operates strictly as a synchronized local mirror. The UI never creates or persists operational state independently of the Core (`KW-ARC-001`, `ADR-0002`).
* **Principle B: Zero Polling & Passive Event Synchronization**  
  Active state updates flow strictly through passive event broadcasts over Tauri IPC bridges (`ipc://status-changed`, `ipc://topology-changed`, `ipc://telemetry-heartbeat`). Continuous `setInterval` or `requestAnimationFrame` polling loops across the IPC boundary are strictly banned (`KW-NF-001`).
* **Principle C: Atomic Action Lifecycle & Ingress Protection**  
  Every user action on the Dashboard transitions through a deterministic four-phase state machine: $\text{Idle} \rightarrow \text{Pending} \rightarrow \text{Confirmed} \mid \text{Failed}$. While in $\text{Pending}$, UI ingress controls (e.g., buttons, switches) enter optimistic loading states and disable duplicate click events to prevent race conditions (`DEC-UI-04`).
* **Principle D: Stale Response Monotonicity (Request Sequence Guardrails)**  
  All asynchronous command invocations and queries carry a monotonically incrementing 64-bit sequence identifier (`requestId: u64`). Any response or event payload arriving with a `requestId` older than the most recently committed transaction is unconditionally dropped.
* **Principle E: Ephemeral UI Lifecycle Invariance**  
  The dashboard UI process (`kinetiwall-ui.exe`) possesses an ephemeral lifecycle: closing the dashboard window releases all frontend memory ($0.0\text{ MB}$ resident RAM). Re-opening the dashboard initiates a single atomic bootstrap fetch (`REQ_GET_DASHBOARD_STATE`) that reconstructs the dashboard state cleanly within $\le 50\text{ ms}$ (`KW-NF-004`).
* **Principle F: Distinct Engine Health vs. Playback State**  
  The system strictly decouples daemon supervisor health (`Active`, `Degraded`, `Critical`, `Unresponsive`) from wallpaper presentation states (`Playing`, `Paused`, `Occluded`, `Muted`, `Error`). A degraded hardware acceleration condition does not falsely register as an engine stoppage (`KW-OBS-002`).
* **Principle G: Resilient Error Containment & Schema Validation**  
  All IPC payloads received by the frontend pass through runtime schema validation (`Zod` / Serde-compatible DTOs). Malformed, partial, or corrupt payloads are trapped at the transport boundary, logging structured telemetry without crashing the React component tree.
* **Principle H: Deterministic Rollback on Failure**  
  If an optimistic UI update fails to receive confirmation from `kinetiwall-core.exe` within the timeout threshold ($5000\text{ ms}$) or returns an error payload, the frontend automatically rolls back to the prior confirmed state and presents an accessible recovery toast.
* **Principle I: Multi-Monitor Topology Decoupling**  
  Per-display playback state is indexed by permanent display hardware identifiers (`displayId`), not volatile system monitor indices. Display connection, disconnection, or resolution reconfigurations update topology slots without desynchronizing unaffected monitors.
* **Principle J: Bounded Retry with Circuit Breaker**  
  If IPC communication with `kinetiwall-core.exe` is interrupted, the UI initiates bounded exponential backoff reconnection attempts ($500\text{ ms}, 1000\text{ ms}, 2000\text{ ms}$) up to 3 times before entering a graceful `Disconnected` state with manual retry options.
* **Principle K: Absolute Memory Leak Prevention**  
  All IPC event listeners registered via Tauri (`listen()`) are bound to React component lifecycles and cleaned up deterministically upon component unmount (`unlisten()`), preventing closure leaks.
* **Principle L: Traceable & Redacted Telemetry**  
  All state mutations, sync failures, and action transitions emit structured, machine-parsable log entries containing correlated transaction IDs (`x-correlation-id`) with all user-identifiable file paths strictly redacted.

---

## 3. Documentation Suite Index

| Deliverable | File Path | Focus & Architectural Scope |
| :--- | :--- | :--- |
| **Index & Principles** | [README.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/dashboard_state_integration/README.md) | Architectural index, 12 core state integration principles, requirements traceability matrix. |
| **Deliverable A** | [DASHBOARD_STATE_INTEGRATION_ARCHITECTURE.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/dashboard_state_integration/DASHBOARD_STATE_INTEGRATION_ARCHITECTURE.md) | State ownership hierarchy (Core vs Zustand vs Derived), bootstrap state acquisition, passive event pipeline, stale response sequence protection, and remounting lifecycle. |
| **Deliverable B** | [ACTION_LIFECYCLE_AND_RELIABILITY.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/dashboard_state_integration/ACTION_LIFECYCLE_AND_RELIABILITY.md) | 4-phase action lifecycle state machine, action ingress protection, optimistic UI rendering, Play/Pause/Mute contracts, and deterministic failure rollback. |
| **Deliverable C** | [STATUS_MAPPING_AND_RECOVERY_STRATEGY.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/dashboard_state_integration/STATUS_MAPPING_AND_RECOVERY_STRATEGY.md) | Centralized status mapping matrix, payload schema validation, bounded retry policies, circuit breakers, and degraded non-destructive fallback rendering. |
| **Deliverable D** | [DASHBOARD_STATE_TEST_PLAN.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/dashboard_state_integration/DASHBOARD_STATE_TEST_PLAN.md) | 4-tier verification test strategy, acceptance criteria matrix (`DSI-001` through `DSI-015`), Vitest store tests, action lifecycle tests, fault injection scenarios. Marked `NOT RUN (DESIGN PHASE)`. |
| **Deliverable E** | [DASHBOARD_STATE_RISK_REGISTER.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/dashboard_state_integration/DASHBOARD_STATE_RISK_REGISTER.md) | Risk analysis (`RSK-DSI-01` through `RSK-DSI-08`), Architecture Decision Records (`DEC-DSI-01` through `DEC-DSI-08`), and open engineering review items. |

---

## 4. Requirements Traceability Matrix

The specifications defined in this document suite directly satisfy and enforce the following system requirements from Phase 1 and 2:

| Requirement ID | Standard Name | How Step 24 Enforces Compliance |
| :--- | :--- | :--- |
| **`KW-F-001`** | Live Wallpaper Playback | Ensures dashboard controls accurately command video/web/shader renderers without desynchronization. |
| **`KW-F-002`** | Multi-Monitor Independence | Binds dashboard display cards to persistent hardware IDs, ensuring isolated per-screen control. |
| **`KW-F-003`** | Wallpaper Assignment & Profiles | Synchronizes profile switches atomically via sequence-guarded IPC contracts. |
| **`KW-F-004`** | Performance Auto-Pause / Occlusion | Accurately maps fullscreen occlusion events to the dashboard status badge without false error states. |
| **`KW-NF-001`** | Idle Resource Utilization | Eliminates frontend polling loops; dashboard relies purely on event-driven push notifications ($0\%\text{ idle CPU}$). |
| **`KW-NF-002`** | Frame Presentation Stability | Prevents dashboard state updates from causing GC pauses or IPC saturation that could drop wallpaper render frames. |
| **`KW-NF-004`** | Cold & Warm Start Latency | Restricts dashboard mount bootstrap to a single consolidated DTO query ($\le 50\text{ ms}$). |
| **`KW-NG-001`** | Pure Rust Core Architecture | Preserves strict separation between Tauri React UI and native Rust daemon (`kinetiwall-core.exe`). |
| **`KW-NG-002`** | Multi-Process Isolation | Ensures that closing `kinetiwall-ui.exe` releases all memory with zero side effects on wallpaper players. |
| **`KW-NG-003`** | Win32 API Safety Contracts | Protects native display handles and IPC channels behind typed Rust FFI and validated JSON-RPC DTOs. |
| **`KW-OBS-001`** | Structured Event Logging | Emits correlated diagnostic telemetry across all state mutations with sanitized privacy paths. |
| **`KW-OBS-002`** | Health & Diagnostics Subsystem | Exposes engine health states (Active, Degraded, Critical) transparently on the dashboard status banner. |

