# Multi-Monitor Management & Geometry Test Matrix
**Document ID:** `KW-DSP-005`  
**Deliverable:** E (Multi-Monitor Test Matrix)  
**Parent Document:** [README.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/display_management/README.md)  
**Classification:** Quality Assurance & System Verification Specification  

---

## 1. Test Strategy & Acceptance Framework

Testing multi-monitor display management requires validating physical coordinate math, DPI scaling transformations, and asynchronous OS hotplug events across diverse display topologies.

Testing is organized into four tiers:
1. **Tier 1: Pure Geometric Unit Tests** (Negative coordinate normalizations, union rectangle math, overflow bounds checks).
2. **Tier 2: Identity & Assignment Matching Tests** (EDID SHA-256 resolution, GDI device name fallbacks, identical monitor disambiguation).
3. **Tier 3: Windows Virtual Display Harness** (Automated multi-monitor simulations using the **Windows Indirect Display Driver (IDD)**).
4. **Tier 4: Adversarial Hotplug & Chaos Tests** (Rapid cable disconnect/reconnect bursts, resolution thrashing, sleep/wake cycles).

> [!IMPORTANT]
> All test cases below represent approved verification specifications. Their status is marked as `NOT RUN (DESIGN PHASE)` pending runtime implementation in Phase 4.

---

## 2. Acceptance Criteria Verification Matrix (`DISP-001` through `DISP-015`)

| Test ID | Test Scenario | Primary Target | Acceptance Criteria | Status |
| :--- | :--- | :--- | :--- | :--- |
| **`DISP-001`** | **Complete Monitor Enumeration** | `EnumDisplayMonitors` | All connected physical and virtual monitors are captured in `TopologySnapshot` with accurate pixel bounds and DPI factors. | `NOT RUN (DESIGN PHASE)` |
| **`DISP-002`** | **EDID Identity Invariance** | Persistent Matching | Reordering monitor display ports or GDI names (`\\.\DISPLAY1` $\leftrightarrow$ `\\.\DISPLAY2`) preserves saved user wallpaper assignments. | `NOT RUN (DESIGN PHASE)` |
| **`DISP-003`** | **Negative Coordinate Normalization** | Virtual Desktop Math | Secondary monitor at $[-1920, 0]$ with virtual origin $[-1920, 0]$ maps child HWND coordinates to $[0, 0]$ inside `WorkerW`. | `NOT RUN (DESIGN PHASE)` |
| **`DISP-004`** | **Coordinate Overflow Rejection** | Math Validation | Simulated display with coordinate $X = 35,000$ or negative width triggers `GeometryError::CoordinateOutOfBounds`; zero crashes. | `NOT RUN (DESIGN PHASE)` |
| **`DISP-005`** | **Independent Playback Isolation**| Multi-Player Topology | Two displays running video wallpapers operate independently; killing Player 1 does not drop frames or interrupt Player 2. | `NOT RUN (DESIGN PHASE)` |
| **`DISP-006`** | **Spanning Union Bounding Box** | Spanning Mode Engine | A $5120 \times 1440$ wallpaper spanning dual $2560 \times 1440$ displays renders across the union rectangle $[0, 0, 5120, 1440]$ seamlessly. | `NOT RUN (DESIGN PHASE)` |
| **`DISP-007`** | **Anti-Oscillation Event Coalescing**| 500ms Debounce Timer | Injecting a burst of 10 `WM_DISPLAYCHANGE` messages within 200 ms triggers exactly 1 final reconciliation task. | `NOT RUN (DESIGN PHASE)` |
| **`DISP-008`** | **Stale Generation Discard** | `TopologyGenerationId` | An in-flight reconciliation task tagged with Gen 1 is discarded when Gen 2 arrives before completion. | `NOT RUN (DESIGN PHASE)` |
| **`DISP-009`** | **Resolution Hot-Resize** | `ResizeBuffers` (D3D11) | Changing monitor resolution from 1080p to 1440p resizes player swapchain buffers in $\le 100\text{ ms}$ without restarting player. | `NOT RUN (DESIGN PHASE)` |
| **`DISP-010`** | **Monitor Disconnect Teardown** | Child Process Supervisor| Disconnecting an HDMI monitor terminates its associated player process within $\le 250\text{ ms}$; VRAM drops to $0\text{ MB}$. | `NOT RUN (DESIGN PHASE)` |
| **`DISP-011`** | **Monitor Reconnect Restoration** | EDID Hash Match | Reconnecting a previously disconnected monitor restores its assigned wallpaper within $\le 500\text{ ms}$. | `NOT RUN (DESIGN PHASE)` |
| **`DISP-012`** | **Zero Duplicate Players** | Process Registry Mutex | Rapidly toggling display power never spawns more than 1 player process per connected monitor glass. | `NOT RUN (DESIGN PHASE)` |
| **`DISP-013`** | **Mixed-DPI Pixel Integrity** | Per-Monitor V2 DPI | 4K 150% DPI primary + 1080p 100% DPI secondary render pixel-perfect swapchains without DWM bitmap blur or boundary overlap. | `NOT RUN (DESIGN PHASE)` |
| **`DISP-014`** | **Workstation Lock Quiescence** | Power & Session Monitor| `WTS_SESSION_LOCK` halts swapchain `Present` and video decoders; CPU utilization drops to $< 0.1\%$ within $20\text{ ms}$. | `NOT RUN (DESIGN PHASE)` |
| **`DISP-015`** | **Zero OS Topology Modification** | Platform Safety Rule | Wallpaper cloning or spanning causes 0 changes to Windows system display settings, resolution, or registry keys. | `NOT RUN (DESIGN PHASE)` |

---

## 3. Virtual Display CI Test Harness Architecture

To enable continuous integration testing of multi-monitor topologies on headless build servers:

```
+-------------------------------------------------------------------------+
|                  CI Test Pipeline: Windows Server / Win11 VM             |
+-------------------------------------------------------------------------+
|  1. Indirect Display Driver (IDD) Harness:                              |
|     - Spawns Virtual Monitor 1: 3840 x 2160 @ 150% DPI (Primary)        |
|     - Spawns Virtual Monitor 2: 1920 x 1080 @ 100% DPI (Left: -1920, 0) |
|  2. Geometry Validator:                                                 |
|     - Evaluates calculate_workerw_bounds across virtual desktop         |
|     - Verifies non-negative bounds [0, 0] and [1920, 0]                 |
|  3. Hotplug Injector:                                                   |
|     - Disables Virtual Monitor 2 via IDD API                            |
|     - Asserts WM_DISPLAYCHANGE received & Player 2 terminated in 250ms  |
|     - Re-enables Virtual Monitor 2                                      |
|     - Asserts Player 2 respawned and matched via EDID hash              |
+-------------------------------------------------------------------------+
```

