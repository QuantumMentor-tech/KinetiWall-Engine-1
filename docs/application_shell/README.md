# Application Shell, Navigation Architecture & Responsive Layout Specification
**Document ID:** `KW-SHL-000`  
**Phase:** 4 (UI/UX Design System & Dashboard)  
**Step:** 22 of 90  
**Status:** Approved Architectural Specification  
**Classification:** Frontend Application Shell, Navigation & Layout Standard  

---

## 1. Document Overview and Scope

This document suite establishes the authoritative engineering specification for KinetiWall's **Root Application Shell, Navigation Architecture, Content Viewport Layout, Responsive Desktop Breakpoints, and View Lifecycle Management**.

The user interface of KinetiWall (`kinetiwall-ui.exe`) operates under strict architectural requirements established in earlier roadmap phases:
1. **Ephemeral Process Lifecycle:** In conformance with `KW-NG-001` and `ADR-0002`, `kinetiwall-ui.exe` is an on-demand process. When the user closes the window, the UI process executes `ExitProcess(0)` and terminates completely, dropping to **$0.0\text{ MB}$ resident RAM**. The shell must initialize rapidly ($\le 350\text{ ms}$ to first interactive frame) without launching duplicate Core instances or blocking active wallpaper playback.
2. **Strict Presentation vs Native Separation:** The application shell owns visual layout, view routing, active-state indication, and responsive window adaptation. It never directly executes Win32 APIs, manages OS processes, or manipulates Windows Explorer `WorkerW` surfaces. All interactions route through typed Tauri IPC bridges (`@tauri-apps/api`) to `kinetiwall-core.exe`.
3. **Approved Neon Glass Design Language:** Top-level layout and navigation adhere strictly to the established cyber-aesthetic: deep void background (`#0A0B10`), acrylic glass navigation sidebar (`#12141F`, `backdrop-filter: blur(16px)`), vibrant neon cyan active accents (`#00F0FF`), crisp border delineations (`#1F2338`), and high-contrast typography.
4. **Desktop Responsive Usability:** Optimized for desktop window sizes ranging from a compact minimum viewport ($960 \times 600\text{ px}$) to full 4K UHD desktop displays ($3840 \times 2160\text{ px}$), guaranteeing that navigation controls, library grids, and monitor canvases adapt without clipping or horizontal scrollbars.

---

## 2. Core Shell Architecture Principles (Principles A through L)

The application shell and navigation architecture is governed by twelve engineering principles:

* **Principle A: Single Authoritative View State**  
  Exactly one primary navigation destination is active at any given time. The visual active indicator strictly mirrors the authoritative view state managed in the client Zustand router store (`src-ui/stores/routerStore.ts`).
* **Principle B: Ephemeral Process Decoupling & Playback Continuity**  
  Switching views, resizing the window, or closing the UI must **never** interrupt, pause, or restart active wallpaper playback on connected monitors. Wallpaper players run in isolated subprocesses supervised exclusively by Core.
* **Principle C: Centralized Navigation Registry**  
  All navigation destinations, route keys, user-facing labels, Lucide icons, and availability flags are declared in a centralized, type-safe registry (`src-ui/routes.ts`). Hardcoding route links or active-state logic across arbitrary components is prohibited.
* **Principle D: Honest Availability & Unimplemented State Guardrails**  
  Unimplemented, deferred, or disabled navigation destinations (e.g. cloud workshops, remote sharing) must never silently route to unrelated views or simulate functionality. They must be styled with explicit disabled indicators (`opacity-40 cursor-not-allowed`) and accessible tooltips.
* **Principle E: Non-Destructive View Switching**  
  Navigating between views (e.g. from Library to Display Settings and back) must preserve in-memory user context: search filter strings, scroll positions, and uncommitted property slider adjustments must not be discarded.
* **Principle F: Clean Event Subscription Lifecycle**  
  Views and components mounted inside the content viewport must clean up all IPC listeners, timers, and event subscriptions upon unmount. Re-navigating to a view must never create duplicate event handlers or memory leaks.
* **Principle G: Bounded Responsive Layout (960px to 4K)**  
  The application shell enforces a minimum desktop window width of $960\text{ px}$ and minimum height of $600\text{ px}$. Content viewports utilize fluid flexbox/grid containers that adapt seamlessly across standard (1080p), high-DPI (1440p/4K), and multi-window snapping layouts.
* **Principle H: Uncompromising WCAG 2.1 AA Keyboard Accessibility**  
  The navigation sidebar and content viewport provide a predictable tab sequence (`tabIndex={0}`), visible 2px neon cyan focus rings (`#00F0FF`), logical ARIA landmark roles (`role="navigation"`, `role="main"`, `aria-current="page"`), and full keyboard activation (`Enter`, `Space`).
* **Principle I: Anti-Thrashing Navigation Invariance**  
  Clicking or activating the currently active navigation item must be a no-op: it must not trigger view re-mounting, network/IPC refetching, or visual flicker.
* **Principle J: Top-Level Error Boundary & View Containment**  
  Rendering errors or exceptions within an active view must be caught by a top-level React Error Boundary. A crash in the Library view must display a graceful in-viewport error card with a "Reset View" action without crashing the global application shell.
* **Principle K: Purposeful Motion & Reduced-Motion Respect**  
  Page transition animations must be swift and subtle ($\le 150\text{ ms}$ fade-in). When the user has enabled "Turn off animations in Windows" (`prefers-reduced-motion: reduce`), all transitions are eliminated instantly.
* **Principle L: Traceable & Redacted Telemetry**  
  View navigation milestones and shell error states emit structured diagnostic events. Sensitive file paths, user search terms, and private window titles are strictly redacted.

---

## 3. Documentation Suite Index

| Deliverable | File Path | Focus & Architectural Scope |
| :--- | :--- | :--- |
| **Index & Principles** | [README.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/application_shell/README.md) | Architectural index, 12 core shell principles, requirements traceability matrix. |
| **Deliverable A** | [APPLICATION_SHELL_ARCHITECTURE.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/application_shell/APPLICATION_SHELL_ARCHITECTURE.md) | Root layout structure, AppShell component design, header bar, persistent navigation rail, content viewport, and IPC boundary rules. |
| **Deliverable B** | [NAVIGATION_ROUTING_SPECIFICATION.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/application_shell/NAVIGATION_ROUTING_SPECIFICATION.md) | Centralized navigation configuration, TypeScript contracts, route state machine, honest availability states, and view lifecycle preservation. |
| **Deliverable C** | [RESPONSIVE_DESKTOP_LAYOUT.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/application_shell/RESPONSIVE_DESKTOP_LAYOUT.md) | Desktop breakpoint definitions (Compact, Standard, Wide, Ultra-Wide), coordinate math, Windows snap layout compatibility, and high-DPI scaling. |
| **Deliverable D** | [APPLICATION_SHELL_TEST_PLAN.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/application_shell/APPLICATION_SHELL_TEST_PLAN.md) | 15-point acceptance criteria matrix (`SHL-001` through `SHL-015`), Vitest unit tests, Playwright layout tests, keyboard focus audits, and CI integration. |
| **Deliverable E** | [APPLICATION_SHELL_RISK_REGISTER.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/application_shell/APPLICATION_SHELL_RISK_REGISTER.md) | Risk register (`RSK-SHL-01..08`), Architecture Decision Records (`DEC-SHL-01..08`), and open engineering review items. |

---

## 4. Requirements Traceability Matrix

| Requirement / Decision | Description | Governing Specification Section |
| :--- | :--- | :--- |
| **`KW-F-001`** | Native desktop shell integration & tray navigation | `APPLICATION_SHELL_ARCHITECTURE.md` §3 |
| **`KW-F-004`** | Local wallpaper library management & browsing shell | `NAVIGATION_ROUTING_SPECIFICATION.md` §2 |
| **`KW-F-007`** | Multi-monitor independent display configuration view | `NAVIGATION_ROUTING_SPECIFICATION.md` §2 |
| **`KW-NF-001`** | CPU utilization limits ($\le 1.0\%$ idle background overhead) | `APPLICATION_SHELL_ARCHITECTURE.md` §4 |
| **`KW-NF-002`** | Memory footprint targets ($\le 120\text{ MB}$ active UI, $0.0\text{ MB}$ closed UI) | `APPLICATION_SHELL_ARCHITECTURE.md` §2 |
| **`KW-NG-001`** | UI ephemeral memory guarantee ($0\text{ MB}$ resident footprint) | `APPLICATION_SHELL_ARCHITECTURE.md` §2 |
| **`ADR-0002`** | Isolated multi-process topology (Core, UI, Per-Display Players) | `APPLICATION_SHELL_ARCHITECTURE.md` §2 |
| **`DEC-UI-01`** | Deterministic process termination on window close (`ExitProcess(0)`) | `APPLICATION_SHELL_ARCHITECTURE.md` §2 |
| **`DEC-SHL-01..08`**| Dedicated application shell, navigation, and layout decisions | `APPLICATION_SHELL_RISK_REGISTER.md` §2 |

