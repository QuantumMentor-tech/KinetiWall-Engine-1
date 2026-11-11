# Display Enumeration, Multi-Monitor Management & DPI Coordination Specification
**Document ID:** `KW-DSP-000`  
**Phase:** 4 (Windows Platform Integration & Desktop Hosting)  
**Step:** 18 of 90  
**Status:** Approved Architectural Specification  
**Classification:** Windows Display Systems & Multi-Monitor Standard  

---

## 1. Document Overview and Scope

This document suite establishes the authoritative engineering specification for **Display Discovery, Monitor Identity Persistence, Multi-Monitor Layout Modes, Virtual Desktop Geometry, Per-Monitor DPI Coordination, and Display-Change Reconciliation** in the **KinetiWall** desktop live wallpaper engine.

Multi-monitor systems under Windows 10 and Windows 11 present complex platform characteristics:
1. **Dynamic Virtual Desktop Topology:** Monitors may be arranged side-by-side, vertically stacked, or positioned with negative virtual-screen coordinates (e.g. secondary display placed to the left of the primary display at $X = -1920$).
2. **Mixed Resolutions & Refresh Rates:** Simultaneous operation across 4K UHD ($3840 \times 2160$ at 60Hz), QHD ($2560 \times 1440$ at 144Hz), and FHD ($1920 \times 1080$ at 240Hz).
3. **Heterogeneous Per-Monitor DPI Scaling:** Combining 150% scaling (primary display) with 100% or 125% scaling (secondary displays).
4. **Volatile OS Enumeration Order:** Win32 device indices (`\\.\DISPLAY1`, `\\.\DISPLAY2`) frequently reorder after driver restarts, dock/undock events, or sleep/wake cycles.
5. **Multi-Monitor Wallpaper Modes:** Supporting independent wallpapers per monitor (`V1.0`), spanning across contiguous displays (`V1.0`), and duplicated/clone presentation (`V1.1`).

This specification establishes an immutable canonical display model, a dual-key persistent identity resolution algorithm (EDID hardware hash + GDI adapter name), robust virtual coordinate math, and an event-driven reconciliation state machine.

---

## 2. Core Display Architecture Principles (Principles A through L)

The display management architecture in KinetiWall is strictly governed by twelve engineering principles:

* **Principle A: Separation of Runtime Topology from Persistent Configuration**  
  Ephemeral Win32 enumeration handles (`HMONITOR`), GDI device indices (`\\.\DISPLAY1`), and process instance IDs must never serve as persistent configuration keys. User wallpaper assignments persist against immutable hardware EDID hashes (`hardware_edid_hash`) and stable device paths.
* **Principle B: Authority of Physical Monitor Bounds**  
  Wallpaper renderers align strictly to physical monitor bounds (`rcMonitor`), never the Windows desktop work area (`rcWork`). The work area excludes space reserved for the taskbar and docked toolbars, whereas wallpapers must span the full physical monitor glass.
* **Principle C: Virtual Desktop Coordinate Normalization**  
  All display and window geometry arithmetic accounts for nonzero virtual screen origins (`SM_XVIRTUALSCREEN`, `SM_YVIRTUALSCREEN`). When embedded within the desktop `WorkerW` hierarchy, child window bounds are normalized relative to the virtual screen top-left origin.
* **Principle D: Single Supervisor Ownership of Display State**  
  `kinetiwall-core.exe` maintains the sole authoritative snapshot of connected displays. Subprocesses (`kinetiwall-player-video.exe`, `kinetiwall-player-web.exe`) and the UI (`kinetiwall-ui.exe`) receive read-only immutable topology snapshots via IPC.
* **Principle E: Independent Failure Isolation**  
  In independent wallpaper mode, a hardware decoder error, display disconnect, or player crash on Monitor A must never interrupt, degrade, or restart wallpaper playback on Monitor B.
* **Principle F: Anti-Oscillation Debouncing & Coalescing**  
  Physical monitor hotplugging or resolution changes emit bursts of rapid OS notifications (`WM_DISPLAYCHANGE`). The supervisor coalesces these events with an unyielding 500 ms hold-off debounce timer before querying the final stable topology.
* **Principle G: Stale Reconciliation Protection**  
  Every display topology query is tagged with a monotonic generation counter (`TopologyGenerationId`). A slow or in-flight reconciliation task tagged with an older generation ID is discarded immediately upon arrival of a newer topology version.
* **Principle H: Native Per-Monitor V2 DPI Awareness**  
  All KinetiWall binaries declare `DPI_AWARENESS_CONTEXT_PER_MONITOR_AWARE_V2`. Window geometries, swapchains, and rendering viewports operate in true physical device pixels, eliminating blurry OS DWM bitmap stretching.
* **Principle I: Non-Destructive Fallback for Disconnected Displays**  
  When an assigned monitor is disconnected or powered off, its configuration is retained intact in the database. The supervisor suspends rendering for that monitor slot without deleting or reassigning the user's wallpaper preference.
* **Principle J: Zero Modification of System Display Settings**  
  KinetiWall must never alter the user's Windows display resolution, refresh rate, orientation, or OS topology. Wallpaper cloning and spanning are implemented entirely inside the application's rendering pipeline.
* **Principle K: Bounded Geometry Validation & Overflow Protection**  
  All rectangle calculations (unions, intersections, scaling transformations) execute with checked integer arithmetic (`checked_add`, `checked_sub`), rejecting coordinates exceeding $\pm 32,767$ pixels to prevent integer overflow.
* **Principle L: Passive Event-Driven Monitoring (Zero Polling)**  
  Display monitoring relies exclusively on passive Win32 event hooks (`WM_DISPLAYCHANGE`, `GUID_MONITOR_POWER_ON`, `TaskbarCreated`), guaranteeing $\le 0.05\%$ background CPU overhead when idle.

---

## 3. Documentation Suite Index

| Deliverable | File Path | Focus & Architectural Scope |
| :--- | :--- | :--- |
| **Index & Principles** | [README.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/display_management/README.md) | Architectural index, 12 core display principles, requirements traceability matrix. |
| **Deliverable A** | [DISPLAY_MANAGEMENT_ARCHITECTURE.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/display_management/DISPLAY_MANAGEMENT_ARCHITECTURE.md) | Canonical display model, subsystem boundaries, enumeration strategy (`EnumDisplayMonitors` + `GetMonitorInfoW`), and multi-monitor rendering modes. |
| **Deliverable B** | [MONITOR_IDENTITY_AND_ASSIGNMENT.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/display_management/MONITOR_IDENTITY_AND_ASSIGNMENT.md) | Persistent identity strategy, EDID hashing, matching algorithms, handling missing/ambiguous monitors, reconnection rules, and privacy. |
| **Deliverable C** | [DISPLAY_GEOMETRY_AND_DPI.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/display_management/DISPLAY_GEOMETRY_AND_DPI.md) | Coordinate systems, negative virtual coordinate math, rectangle union/intersection math, Per-Monitor V2 DPI scaling, and overflow prevention. |
| **Deliverable D** | [DISPLAY_RECONCILIATION_POLICY.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/display_management/DISPLAY_RECONCILIATION_POLICY.md) | `WM_DISPLAYCHANGE` handling, 500ms debounce coalescing, generation ID sequencing, dynamic player resizing, and degraded fallback states. |
| **Deliverable E** | [MULTI_MONITOR_TEST_MATRIX.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/display_management/MULTI_MONITOR_TEST_MATRIX.md) | Comprehensive verification matrix (`DISP-001` through `DISP-015`), virtual display test fixtures (IDD), lifecycle tests, and performance benchmarks. |
| **Deliverable F** | [DISPLAY_MANAGEMENT_RISK_REGISTER.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/display_management/DISPLAY_MANAGEMENT_RISK_REGISTER.md) | Risk register (`RSK-DSP-01..08`), Architecture Decision Records (`DEC-DSP-01..08`), and open engineering review items. |

---

## 4. Requirements Traceability Matrix

| Requirement / Decision | Description | Governing Specification Section |
| :--- | :--- | :--- |
| **`KW-F-001`** | Native desktop shell integration & display targeting | `DISPLAY_MANAGEMENT_ARCHITECTURE.md` §3 |
| **`KW-F-007`** | Multi-monitor independent display configuration & geometry | `DISPLAY_MANAGEMENT_ARCHITECTURE.md` §4 |
| **`KW-F-008`** | Power-state and fullscreen hardware occlusion detection | `DISPLAY_RECONCILIATION_POLICY.md` §4 |
| **`KW-NF-001`** | CPU utilization limits ($\le 1.0\%$ idle background overhead) | `DISPLAY_RECONCILIATION_POLICY.md` §5 |
| **`KW-NF-006`** | Graceful shutdown and rapid exit response ($\le 250\text{ ms}$) | `DISPLAY_RECONCILIATION_POLICY.md` §3 |
| **`ADR-0002`** | Isolated multi-process topology (Core, UI, Per-Display Players) | `DISPLAY_MANAGEMENT_ARCHITECTURE.md` §2 |
| **`DEC-DATA-02`** | Configuration schema with monitor map & EDID hash persistence | `MONITOR_IDENTITY_AND_ASSIGNMENT.md` §2 |
| **`DEC-DSK-05`** | Per-monitor DPI & virtual coordinate normalization | `DISPLAY_GEOMETRY_AND_DPI.md` §2 |
| **`DEC-DSP-01..08`**| Dedicated display management, spanning, and identity decisions | `DISPLAY_MANAGEMENT_RISK_REGISTER.md` §2 |

