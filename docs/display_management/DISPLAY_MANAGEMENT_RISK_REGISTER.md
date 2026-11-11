# Display Management Risk Register & Architectural Decisions
**Document ID:** `KW-DSP-006`  
**Deliverable:** F (Display Management Risk Register)  
**Parent Document:** [README.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/display_management/README.md)  
**Classification:** Core System Architecture / Operational Reliability Standard  

---

## 1. Display Management Risk Register

Multi-monitor management on Windows introduces subtle edge cases in display topology, graphics drivers, and hardware handshakes:

| Risk ID | Risk Summary & Failure Mode | Prob. | Impact | Severity | Architecture Mitigation & Prevention Strategy |
| :--- | :--- | :---: | :---: | :---: | :--- |
| **`RSK-DSP-01`** | **Wallpaper Swapping on Driver Reload**<br>Windows re-enumerates GDI display indices in reverse order after driver crash or reboot, swapping wallpapers. | High | Med | **HIGH** | **Dual-Key EDID Hashing (`DEC-DSP-01`)**: Wallpaper assignments persist against SHA-256 EDID hardware hashes, completely decoupled from volatile `\\.\DISPLAY#` names. |
| **`RSK-DSP-02`** | **Reconciliation Thrashing on Hotplug**<br>Plugging an HDMI display emits 10 rapid `WM_DISPLAYCHANGE` messages, causing continuous player spawn/kill loops. | High | Med | **HIGH** | **500ms Anti-Oscillation Debounce (`DEC-DSP-02`)**: Display changes are coalesced with a monotonic 500ms hold-off timer before triggering reconciliation. |
| **`RSK-DSP-03`** | **Stale Topology Overwrites Newer State**<br>A delayed async reconciliation task completes after a newer monitor topology has already established, corrupting state. | Med | High | **HIGH** | **Generation Counter Gating (`DEC-DSP-03`)**: Every display snapshot is tagged with `TopologyGenerationId`; outdated generation tasks are discarded immediately. |
| **`RSK-DSP-04`** | **Negative Coordinate Clipping in WorkerW**<br>Secondary monitor placed to the left of primary has negative coordinates ($X < 0$), causing child HWND to render offscreen. | Med | High | **HIGH** | **Virtual Screen Offset Normalization (`DEC-DSP-04`)**: Child window coordinates are normalized by subtracting `SM_XVIRTUALSCREEN` and `SM_YVIRTUALSCREEN`. |
| **`RSK-DSP-05`** | **Identical Dual Monitor Collision**<br>Two identical monitors of the same model report identical EDID serial numbers (`00000000`), confusing assignments. | Med | Med | **MEDIUM** | **GPU Output Port Disambiguation (`DEC-DSP-05`)**: Identical EDID hashes are disambiguated by appending the hardware GPU adapter video output port ID. |
| **`RSK-DSP-06`** | **Integer Overflow on High-Res Multi-Display**<br>Combining multiple 4K/8K displays produces coordinates exceeding 16-bit Win32 coordinate boundaries, crashing engine. | Low | High | **HIGH** | **Checked Integer Arithmetic & 32k Limits (`DEC-DSP-06`)**: All geometry calculations use `checked_add` and reject coordinates exceeding $\pm 32,767$ pixels. |
| **`RSK-DSP-07`** | **Duplicate Player Spawning Race**<br>Rapid display sleep/wake cycles trigger concurrent player spawns for the same display slot, leaking processes. | Med | High | **HIGH** | **Single-Player Slot Mutex & Job Objects (`DEC-DSP-07`)**: Core enforces exclusive single-player ownership per display slot and terminates old player before spawning. |
| **`RSK-DSP-08`** | **DWM Scaling Blur on Mixed-DPI**<br>4K 150% + 1080p 100% displays suffer blurry bitmap stretching due to incorrect process DPI awareness. | Med | Med | **HIGH** | **Mandatory Per-Monitor V2 DPI (`DEC-DSP-08`)**: All binaries declare `DPI_AWARENESS_CONTEXT_PER_MONITOR_AWARE_V2`, rendering in 1:1 physical device pixels. |

---

## 2. Architecture Decision Records (Display Management: `DEC-DSP-01` to `08`)

The following formal decisions govern all display enumeration and multi-monitor management implementations across KinetiWall:

### `DEC-DSP-01`: Authoritative Adoption of Dual-Key EDID Persistent Identity
* **Context:** Win32 GDI device names (`\\.\DISPLAY1`) and `HMONITOR` handles are volatile and change between reboots.
* **Decision:** Persist wallpaper assignments against cryptographic SHA-256 hashes of the hardware EDID block (`hardware_edid_hash`), falling back to SetupAPI device instance paths.
* **Consequences:** Eliminates wallpaper swapping bugs permanently across reboots, port swaps, and driver updates.

### `DEC-DSP-02`: Monotonic 500ms Anti-Oscillation Debounce for Display Changes
* **Context:** Monitor hotplugging emits bursts of rapid `WM_DISPLAYCHANGE` events that can cause rendering thrashing.
* **Decision:** Core holds off display reconciliation until 500 ms of silence has elapsed following the last `WM_DISPLAYCHANGE` message.
* **Consequences:** Guarantees that DirectX swapchains and video decoders reconfigure exactly once per physical user action.

### `DEC-DSP-03`: Monotonic `TopologyGenerationId` for Stale Reconciliation Protection
* **Context:** Asynchronous display reconciliation tasks can complete out of order during rapid display changes.
* **Decision:** Tag every topology query with a monotonic `u64` generation ID. In-flight tasks check if their generation matches `current_generation`; if stale, they exit immediately.
* **Consequences:** Eliminates race conditions and prevents stale display states from corrupting active playback.

### `DEC-DSP-04`: Mandatory Virtual Screen Normalization for `WorkerW` Child Windows
* **Context:** Win32 global screen coordinates can be negative, but child window coordinates inside `WorkerW` must be relative to the `WorkerW` top-left origin.
* **Decision:** Child window bounds are normalized by subtracting `SM_XVIRTUALSCREEN` and `SM_YVIRTUALSCREEN` from physical monitor bounds.
* **Consequences:** Ensures child render windows have clean, non-negative coordinates, completely preventing offscreen clipping.

### `DEC-DSP-05`: GPU Output Port Disambiguation for Identical Dual Monitors
* **Context:** Budget monitors of identical make and model often share dummy serial numbers (`00000000`) in their EDID.
* **Decision:** When identical EDID hashes are detected, append the GPU adapter output port instance path to create a unique persistent key.
* **Consequences:** Guarantees unambiguous wallpaper assignment even when identical monitors are connected.

### `DEC-DSP-06`: Checked Integer Arithmetic with 32k Win32 Coordinate Clamping
* **Context:** Multi-monitor virtual desktop extents can exceed integer bounds if corrupted or virtualized.
* **Decision:** Execute all coordinate math with `checked_sub` and `checked_add`, rejecting coordinates $> 32,767$ pixels with `GeometryError::CoordinateOutOfBounds`.
* **Consequences:** Prevents integer overflow vulnerabilities and undefined behavior in native Win32 window-positioning APIs.

### `DEC-DSP-07`: Single-Player Mutex per Monitor Slot with Job Object Containment
* **Context:** Concurrent display events could race to spawn duplicate renderers for the same physical glass.
* **Decision:** Core maintains an exclusive lock per monitor slot, requiring full termination of an existing player before spawning a replacement, with fallback Job Object containment.
* **Consequences:** Guarantees zero duplicate player processes and zero zombie renderers during hotplug events.

### `DEC-DSP-08`: Universal Per-Monitor V2 DPI Declaration
* **Context:** Mixed-DPI setups suffer from blurry rendering if processes declare legacy system DPI awareness.
* **Decision:** Declare `DPI_AWARENESS_CONTEXT_PER_MONITOR_AWARE_V2` across all KinetiWall executables; swapchains allocate 1:1 physical pixels.
* **Consequences:** Delivers crisp, pixel-perfect video and web wallpapers across high-DPI (4K) and standard-DPI (1080p) displays without DWM scaling artifacts.

---

## 3. Open Engineering Review Items

| Item ID | Description | Current Architectural Status | Target Phase |
| :---: | :--- | :--- | :---: |
| **`REV-DSP-01`** | **Dynamic Refresh Rate (VRR / G-Sync) Interaction:** Evaluate whether variable refresh rate monitors require disabling VSync presentation in mpv to avoid desktop stutter. | VSync presentation enabled by default; evaluating DWM interaction on VRR displays. | Phase 4 (Runtime Verification) |
| **`REV-DSP-02`** | **Physical Bezel Pixel Compensation:** Design mathematical pixel offsets to correct visual alignment across physical monitor bezels in spanning mode. | Basic bounding union implemented in V1.0; bezel compensation scheduled for V1.1. | Phase 4 (Runtime Verification) |
| **`REV-DSP-03`** | **HDR Tone Mapping per Display:** Coordinate HDR10 swapchains when one display is HDR-capable and a secondary display is SDR. | Evaluated under D3D11 architecture; swapchain formats configured independently per display player. | Phase 4 (Runtime Verification) |

