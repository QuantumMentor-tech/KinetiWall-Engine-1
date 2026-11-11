# Dashboard & Home Screen Experience Specification
**Document ID:** `KW-DSH-000`  
**Phase:** 4 (UI/UX Design System & Dashboard)  
**Step:** 23 of 90  
**Status:** Approved Architectural Specification  
**Classification:** Frontend Application Experience / Dashboard & Overview Standard  

---

## 1. Document Overview and Scope

This document suite establishes the authoritative engineering specification for KinetiWall's **Primary Dashboard & Home Screen Overview Experience** (`src-ui/views/DashboardView.tsx`).

The Dashboard acts as the primary landing surface for `kinetiwall-ui.exe` upon user launch or system tray activation. It provides a cohesive, truthful, and high-performance overview answering three essential user questions:
1. **What is KinetiWall doing right now?** Displaying active wallpaper title, preview thumbnail, engine status (Active, Paused, Occluded, Stopped), active display count, and real-time presentation framerate ($60\text{ FPS}$).
2. **What can I do next?** Exposing contextual quick actions: Play/Pause toggle, Mute/Unmute audio, "Browse Library", "Configure Displays", and "Engine Settings".
3. **Is there anything requiring attention?** Providing unambiguous diagnostic status banners: alerting the user if graphics acceleration encountered a recoverable device reset, if a display was disconnected, or if battery conservation is throttling framerates.

In strict conformance with KinetiWall's architectural charter (`KW-NG-001`, `ADR-0002`, `DEC-UI-01`), the Dashboard operates within an **ephemeral process boundary**: closing the window terminates `kinetiwall-ui.exe` ($0.0\text{ MB}$ resident RAM) without interrupting active wallpaper rendering.

---

## 2. Core Dashboard Principles (Principles A through L)

The dashboard architecture in KinetiWall is strictly governed by twelve engineering principles:

* **Principle A: Truthful & Reliable Operational State**  
  Every metric, status badge, and wallpaper detail displayed on the Dashboard must originate from authoritative IPC queries (`kinetiwall-core.exe`). The dashboard must never fabricate sample wallpapers, generate synthetic FPS readings, or simulate successful operations.
* **Principle B: Zero Interference with Wallpaper Playback**  
  Mounting the Dashboard, navigating between sections, or closing the UI must **never** restart, pause, or interrupt active wallpaper playback on connected monitors (`DEC-SHL-02`).
* **Principle C: Ephemeral Lifecycle & Zero Polling**  
  The Dashboard acquires its initial state via a single IPC bootstrap query (`REQ_GET_DASHBOARD_STATE`) upon mount. Ongoing updates arrive through passive Tauri IPC event pushes (`ipc://status-changed`). Continuous background polling loops are strictly prohibited (`KW-NF-001`).
* **Principle D: Clear Visual Hierarchy & Glanceable Layout**  
  The Dashboard adheres to a 4-quadrant layout: (1) Header Greeting & Global Status, (2) Active Wallpaper Hero Panel, (3) Quick Action Controls, and (4) Display Topology & Engine Health Cards.
* **Principle E: Atomic Design System Reusability**  
  Every dashboard element reuses the foundational primitives (`Button`, `Badge`, `Switch`, `Slider`, `ConfirmationDialog`) and design tokens established in Steps 20 and 21. Hardcoding parallel styling systems is prohibited.
* **Principle F: Distinct & Useful Empty States**  
  If no wallpaper is currently assigned, the Dashboard renders an engaging onboarding card ("No Active Wallpaper") with an immediate one-click call-to-action ("Browse Wallpaper Library").
* **Principle G: Resilient Error Containment & Partial Failure**  
  If the preview thumbnail of an active wallpaper is missing or corrupt, the Hero panel renders a graceful SVG fallback card without crashing or blocking the Play/Pause controls.
* **Principle H: Responsive Adaptation ($960\text{ px}$ to 4K)**  
  The Dashboard layout dynamically reorganizes from a stacked single-column view on compact desktop windows ($960\text{ px}$) to an expansive multi-column dashboard on 4K displays ($\ge 2400\text{ px}$).
* **Principle I: Instant Feedback with Action Ingress Disabling**  
  Triggering an asynchronous action (e.g. toggling Pause or Mute) instantly reflects an optimistic loading state and suppresses duplicate clicks until Core confirmation arrives (`DEC-UI-04`).
* **Principle J: Absolute Keyboard Accessibility (WCAG 2.1 AA)**  
  Every action button, card trigger, and settings shortcut provides a 2px neon cyan focus ring (`#00F0FF`), logical tab sequence, and full keyboard operability (`Enter`, `Space`).
* **Principle K: Strict IPC Boundary & Data Sanitization**  
  Wallpaper titles and author metadata are rendered strictly as escaped text nodes, eliminating XSS injection hazards (`DEC-SEC-05`). Dashboard components never access raw Win32 handles or native OS pipes directly.
* **Principle L: Traceable & Redacted Telemetry**  
  Dashboard navigation events and operational error banners emit structured diagnostic records with sensitive personal paths strictly redacted (`KW-OBS-001`).

---

## 3. Documentation Suite Index

| Deliverable | File Path | Focus & Architectural Scope |
| :--- | :--- | :--- |
| **Index & Principles** | [README.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/dashboard_experience/README.md) | Architectural index, 12 core dashboard principles, requirements traceability matrix. |
| **Deliverable A** | [DASHBOARD_ARCHITECTURE_AND_LAYOUT.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/dashboard_experience/DASHBOARD_ARCHITECTURE_AND_LAYOUT.md) | 4-quadrant layout blueprint, Active Wallpaper Hero Card, Quick Action Grid, Display Health Summary, and responsive breakpoints. |
| **Deliverable B** | [DATA_INTEGRATION_AND_STATUS_MODEL.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/dashboard_experience/DATA_INTEGRATION_AND_STATUS_MODEL.md) | Authoritative IPC data contracts (`DashboardBootstrapDTO`), engine status mappings, playback state derivation, and passive event synchronization. |
| **Deliverable C** | [EMPTY_ERROR_AND_ONBOARDING_STATES.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/dashboard_experience/EMPTY_ERROR_AND_ONBOARDING_STATES.md) | Visual design and user guidance for No Active Wallpaper, Engine Disconnected, Missing Assets, and Degraded Driver States. |
| **Deliverable D** | [DASHBOARD_EXPERIENCE_TEST_PLAN.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/dashboard_experience/DASHBOARD_EXPERIENCE_TEST_PLAN.md) | 15-point acceptance criteria matrix (`DSH-001` through `DSH-015`), Vitest unit tests, Playwright UI tests, keyboard focus audits, and CI integration. |
| **Deliverable E** | [DASHBOARD_EXPERIENCE_RISK_REGISTER.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/dashboard_experience/DASHBOARD_EXPERIENCE_RISK_REGISTER.md) | Risk register (`RSK-DSH-01..08`), Architecture Decision Records (`DEC-DSH-01..08`), and open engineering review items. |

---

## 4. Requirements Traceability Matrix

| Requirement / Decision | Description | Governing Specification Section |
| :--- | :--- | :--- |
| **`KW-F-001`** | Core engine initialization and native desktop shell integration | `DASHBOARD_ARCHITECTURE_AND_LAYOUT.md` §2 |
| **`KW-F-004`** | Local wallpaper library management & dashboard overview | `DASHBOARD_ARCHITECTURE_AND_LAYOUT.md` §2 |
| **`KW-F-006`** | Performance metrics visualization (FPS, VRAM, CPU) | `DATA_INTEGRATION_AND_STATUS_MODEL.md` §2 |
| **`KW-NF-001`** | CPU utilization limits ($\le 1.0\%$ idle background overhead) | `DATA_INTEGRATION_AND_STATUS_MODEL.md` §3 |
| **`KW-NF-002`** | Memory footprint targets ($\le 120\text{ MB}$ active UI, $0.0\text{ MB}$ closed UI) | `DASHBOARD_ARCHITECTURE_AND_LAYOUT.md` §1 |
| **`KW-NG-001`** | UI ephemeral memory guarantee ($0\text{ MB}$ resident footprint) | `DASHBOARD_ARCHITECTURE_AND_LAYOUT.md` §1 |
| **`ADR-0002`** | Isolated multi-process topology (Core, UI, Per-Display Players) | `DASHBOARD_ARCHITECTURE_AND_LAYOUT.md` §1 |
| **`DEC-UI-04`** | Instant visual feedback & ingress disabling for actions | `DASHBOARD_ARCHITECTURE_AND_LAYOUT.md` §3 |
| **`DEC-DSH-01..08`**| Dedicated dashboard layout, truthful data, and empty-state decisions | `DASHBOARD_EXPERIENCE_RISK_REGISTER.md` §2 |

