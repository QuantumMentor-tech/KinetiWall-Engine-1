# Windows Desktop Integration Architecture & Compatibility Strategy
**Document ID:** `KW-DSK-000`  
**Phase:** 4 (Windows Platform Integration & Desktop Hosting)  
**Step:** 16 of 90  
**Status:** Approved Architectural Specification  
**Classification:** Windows Platform Integration & Desktop Shell Standard  

---

## 1. Document Overview and Scope

This document suite establishes the authoritative engineering specification for integrating **KinetiWall** live wallpaper renderers directly into the Windows desktop environment.

Live wallpaper applications on Microsoft Windows occupy a delicate architectural niche: they must project hardware-accelerated video streams, interactive 3D WebGL animations, and shaders behind desktop icons (`SysListView32`), while maintaining zero disruption to:
1. Standard desktop icon selection, dragging, and hover effects.
2. Shell context menus (right-click desktop menu).
3. System taskbar, Start menu, system tray, and Action Center z-orders.
4. Active top-level and fullscreen foreground applications (games, IDEs, office suites).
5. Windows shell stability across Explorer crashes, resolution changes, and multi-monitor hotplugging.

This specification analyzes both officially documented Win32 window-management contracts and undocumented Windows Shell mechanisms (specifically `Progman`, `WorkerW`, and message `0x052C`). It establishes an evidence-based integration strategy, a 10-state integration state machine, an authoritative window ownership model, multi-monitor coordinate math, and a rigorous compatibility and test matrix.

---

## 2. Core Desktop Integration Principles (Principles A through L)

Desktop integration in KinetiWall is governed by twelve engineering principles:

* **Principle A: Layer Integrity & Non-Inference**  
  Wallpaper windows must be positioned strictly between the desktop wallpaper background surface and the shell desktop icon layer (`SysListView32`). A window handle alone is never proof of correct layer integration; placement must be verified through child window hierarchy traversal.
* **Principle B: Zero Interference with Shell Usability**  
  Normal desktop icon interaction (single-click, double-click, marquee rectangle drag-select, keyboard navigation), context menus, taskbar auto-hide, and Start menu activation must remain 100% operational and unobstructed.
* **Principle C: Transparent Click-Through Input Default**  
  For all non-interactive video wallpaper players, the renderer window must possess extended window styles `WS_EX_TRANSPARENT | WS_EX_NOACTIVATE | WS_EX_LAYERED` to guarantee that mouse clicks pass through completely to the desktop icon listview underneath.
* **Principle D: Clear Demarcation of Undocumented Behavior**  
  The `0x052C` message to `Progman` and the resulting `WorkerW` split are explicitly documented as **undocumented, community-observed Windows Shell internal mechanics**. They must never be treated as guaranteed Win32 API contracts, and must be guarded by robust fallback and validation gates.
* **Principle E: Strict Multi-Process Window Ownership**  
  In conformance with `ADR-0002` and `PROC-01..04`, wallpaper renderers run in isolated subprocesses (`kinetiwall-player-video.exe`, `kinetiwall-player-web.exe`). The player process creates and owns its rendering HWND. Reparenting (`SetParent`) into the shell's `WorkerW` is performed under strict validation by the player process itself upon Core authorization.
* **Principle F: Dual-Key Verification Against PID and Handle Recycling**  
  No window handle (HWND) discovered from the desktop shell is ever manipulated without validating its window class name (`WorkerW` or `Progman`), its owning thread/process ID, and its terminal session identity.
* **Principle G: Resilient Explorer Restart Recovery**  
  When `explorer.exe` crashes or restarts, the integration architecture detects the event via `RegisterWindowMessageW("TaskbarCreated")` and shell process exit observation, gracefully tearing down stale handles and re-anchoring renderers without terminating active player processes or dropping decoder frames.
* **Principle H: Coordinate Precision Across Mixed-DPI & Virtual Screen**  
  Every renderer window geometry is calculated using physical monitor coordinates obtained from `GetMonitorInfoW` and per-monitor DPI scaling (`DPI_AWARENESS_CONTEXT_PER_MONITOR_AWARE_V2`), properly handling negative coordinates and virtual desktop boundaries.
* **Principle I: Observable Failure & Safe Quiescent Fallback**  
  If the shell window hierarchy is missing, ambiguous, or corrupted, the integration layer enters `INTEGRATION_DEGRADED` or `INTEGRATION_FAILED`. The engine drops into a safe stopped state, restoring the native Windows desktop wallpaper rather than creating an unanchored floating window that steals user focus.
* **Principle J: Zero In-Memory Resident Footprint for UI**  
  The configuration GUI (`kinetiwall-ui.exe`) never participates in desktop hosting or window reparenting. Closing the UI window completely terminates `kinetiwall-ui.exe` ($0.0\text{ MB}$ footprint) without impacting desktop integration or active player HWNDs.
* **Principle K: Bounded Attachment & Teardown Budgets**  
  Window discovery, host validation, and reparenting operations must execute in $\le 50\text{ ms}$. Teardown and unparenting during stop or shutdown must execute in $\le 20\text{ ms}$ to comply with the global $250\text{ ms}$ shutdown budget (`KW-NF-006`).
* **Principle L: Traceable & Redacted Telemetry**  
  Every window attachment, reparenting event, DPI change, or Explorer recovery milestone emits structured diagnostics. Raw user window titles or unrelated third-party HWNDs are strictly redacted.

---

## 3. Documentation Suite Index

| Deliverable | File Path | Focus & Architectural Scope |
| :--- | :--- | :--- |
| **Index & Principles** | [README.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/desktop_integration/README.md) | Architectural index, 12 core desktop integration principles, requirements matrix. |
| **Deliverable A** | [WINDOWS_DESKTOP_INTEGRATION_ARCHITECTURE.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/desktop_integration/WINDOWS_DESKTOP_INTEGRATION_ARCHITECTURE.md) | Desktop layer requirements, candidate strategies, decision matrix, initialization protocol, 10-state integration state machine. |
| **Deliverable B** | [WORKERW_AND_SHELL_COMPATIBILITY_RESEARCH.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/desktop_integration/WORKERW_AND_SHELL_COMPATIBILITY_RESEARCH.md) | Deep technical dossier on `Progman`, `WorkerW`, message `0x052C`, documented APIs vs undocumented shell internals, and cross-version behavior. |
| **Deliverable C** | [DESKTOP_WINDOW_OWNERSHIP_MATRIX.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/desktop_integration/DESKTOP_WINDOW_OWNERSHIP_MATRIX.md) | Complete inventory of integration windows, creators, owning threads, parent/child relationships, styles, and cleanup obligations. |
| **Deliverable D** | [DISPLAY_AND_DESKTOP_RECONCILIATION_STRATEGY.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/desktop_integration/DISPLAY_AND_DESKTOP_RECONCILIATION_STRATEGY.md) | Multi-monitor geometry, virtual desktop math, mixed DPI, display hotplugging, resolution changes, and Explorer restart recovery. |
| **Deliverable E** | [WINDOWS_DESKTOP_INTEGRATION_MATRIX.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/desktop_integration/WINDOWS_DESKTOP_INTEGRATION_MATRIX.md) | Compatibility test matrix covering Windows 10/11 builds, multi-monitor topologies, shell lifecycle events, and performance benchmarks. |
| **Deliverable F** | [DESKTOP_INTEGRATION_RISK_REGISTER.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/desktop_integration/DESKTOP_INTEGRATION_RISK_REGISTER.md) | Failure modes, risk assessment (`RSK-DSK-01..08`), Architecture Decision Records (`DEC-DSK-01..08`), and open review items. |

---

## 4. Requirements Traceability Matrix

| Requirement / Decision | Description | Governing Specification Section |
| :--- | :--- | :--- |
| **`KW-F-001`** | Core engine initialization and native desktop shell integration | `WINDOWS_DESKTOP_INTEGRATION_ARCHITECTURE.md` §3 |
| **`KW-F-002`** | High-performance video wallpaper playback (D3D11 / mpv) | `DESKTOP_WINDOW_OWNERSHIP_MATRIX.md` §2 |
| **`KW-F-003`** | Interactive web wallpaper playback (WebView2) | `DESKTOP_WINDOW_OWNERSHIP_MATRIX.md` §2 |
| **`KW-F-007`** | Multi-monitor independent display configuration & geometry | `DISPLAY_AND_DESKTOP_RECONCILIATION_STRATEGY.md` §1 |
| **`KW-F-008`** | Power-state and fullscreen hardware occlusion detection | `WINDOWS_DESKTOP_INTEGRATION_ARCHITECTURE.md` §7 |
| **`KW-NF-001`** | CPU utilization limits ($\le 1.0\%$ idle background) | `DISPLAY_AND_DESKTOP_RECONCILIATION_STRATEGY.md` §4 |
| **`KW-NF-006`** | Graceful shutdown and rapid exit response ($\le 250\text{ ms}$) | `WINDOWS_DESKTOP_INTEGRATION_ARCHITECTURE.md` §6 |
| **`KW-NG-001`** | UI ephemeral memory guarantee ($0\text{ MB}$ resident footprint) | `DESKTOP_WINDOW_OWNERSHIP_MATRIX.md` §1 |
| **`ADR-0002`** | Isolated multi-process topology (Core, UI, Per-Display Players) | `WINDOWS_DESKTOP_INTEGRATION_ARCHITECTURE.md` §2 |
| **`DEC-PRF-08`** | Windows Job Object orphan process containment | `WINDOWS_DESKTOP_INTEGRATION_ARCHITECTURE.md` §2 |
| **`DEC-LFC-08`** | Explicit desktop shell surface restoration on teardown | `WINDOWS_DESKTOP_INTEGRATION_ARCHITECTURE.md` §6 |
| **`DEC-DSK-01..08`** | Dedicated desktop integration decisions covering WorkerW, SetParent | `DESKTOP_INTEGRATION_RISK_REGISTER.md` §2 |

