# Desktop Integration Risk Register & Architectural Decisions
**Document ID:** `KW-DSK-006`  
**Deliverable:** F (Desktop Integration Risk Register)  
**Parent Document:** [README.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/desktop_integration/README.md)  
**Classification:** Core System Architecture / Operational Reliability Standard  

---

## 1. Desktop Integration Risk Register

Integrating real-time rendering pipelines into Windows Explorer's internal window hierarchy introduces technical risks that must be analyzed and defended:

| Risk ID | Risk Summary & Failure Mode | Prob. | Impact | Severity | Architecture Mitigation & Prevention Strategy |
| :--- | :--- | :---: | :---: | :---: | :--- |
| **`RSK-DSK-01`** | **OS Update Breaks Undocumented `0x052C`**<br>A future Windows 11 feature update alters Explorer's response to `0x052C`, removing `WorkerW` split behavior. | Low | High | **CRITICAL** | **Defensive Validation & Safe Fallback**: If `discover_behind_icon_workerw()` fails to find a valid `WorkerW`, integration safely drops to `INTEGRATION_DEGRADED`. Native wallpaper remains untouched; zero crashes. |
| **`RSK-DSK-02`** | **Desktop Icons Occluded / Vanished**<br>Player window reparented into the wrong `WorkerW` or placed at incorrect z-order, covering `SysListView32`. | Med | High | **CRITICAL** | **Two-Pass WorkerW Validation (`DEC-DSK-01`)**: Discovery verifies that candidate `WorkerW` does *not* contain `SHELLDLL_DefView`. Layer placement is audited post-attachment. |
| **`RSK-DSK-03`** | **Explorer Deadlock via Synchronous Calls**<br>Calling `SendMessageW(hProgman, 0x052C)` deadlocks the calling thread if Windows Explorer is unresponsive or hung. | Med | High | **HIGH** | **Mandatory Asynchronous Timeout (`DEC-DSK-02`)**: All shell message dispatches use `SendMessageTimeoutW` with an unyielding $1000\text{ ms}$ timeout and `SMTO_NORMAL`. |
| **`RSK-DSK-04`** | **Stale HWND Manipulation Post-Crash**<br>Explorer crashes; Core attempts to position or unparent an invalid HWND, risking operations on recycled window handles. | High | Med | **HIGH** | **`IsWindow` & Process ID Verification (`DEC-DSK-04`)**: All window handles are re-validated before any API call. Explorer death invalidates cached handles immediately. |
| **`RSK-DSK-05`** | **Input Queue Synchronization Lock**<br>`SetParent` cross-process causes child player message queue to synchronize with Explorer, inducing stutter. | Med | Med | **HIGH** | **Transparent Window Styles (`DEC-DSK-03`)**: Non-interactive player windows are styled with `WS_EX_TRANSPARENT | WS_EX_NOACTIVATE`, instructing Windows to bypass input routing to the child. |
| **`RSK-DSK-06`** | **Negative Virtual Coordinate Clipping**<br>Secondary monitor placed to the left of primary has negative coordinates, causing player window to render offscreen. | Med | Med | **MEDIUM** | **Virtual Screen Offset Normalization (`DEC-DSK-05`)**: Window bounds are normalized by subtracting `SM_XVIRTUALSCREEN` and `SM_YVIRTUALSCREEN` from monitor rects. |
| **`RSK-DSK-07`** | **Black Surface on Player Teardown**<br>Player exits abruptly without restoring shell desktop surface, leaving a black rectangle behind icons. | Med | Med | **HIGH** | **Deterministic Redraw & Unparenting (`DEC-DSK-06`)**: Player invokes `ShowWindow(SW_HIDE)` and `SetParent(NULL)`, followed by `RedrawWindow(hWorkerW, RDW_INVALIDATE)` before exit. |
| **`RSK-DSK-08`** | **Explorer Restart Infinite Loop**<br>Explorer crashes in a loop; Core attempts continuous reconnection, consuming 100% CPU. | Low | High | **HIGH** | **Exponential Backoff Circuit Breaker (`DEC-DSK-07`)**: Maximum 3 reconnect attempts. If Explorer crashes 3 times in 60 seconds, Core suspends reconnection until manual reset. |

---

## 2. Architecture Decision Records (Desktop Hosting: `DEC-DSK-01` to `08`)

The following formal decisions govern all desktop integration and shell hosting implementations across KinetiWall:

### `DEC-DSK-01`: Approval of WorkerW-Based Shell Integration via `0x052C`
* **Context:** Windows desktop wallpaper engines require rendering behind `SysListView32`. No public Win32 API exists for live wallpapers.
* **Decision:** Approve Strategy 2 (`WorkerW` Split via `0x052C`) as the primary integration architecture, guarded by explicit two-pass window enumeration and class validation.
* **Consequences:** Delivers 100% desktop icon compatibility, native context menu functionality, and seamless taskbar integration across Windows 10 and Windows 11. Acknowledges undocumented status and provides safe fallback.

### `DEC-DSK-02`: Mandatory `SendMessageTimeoutW` for All Shell Interactions
* **Context:** Sending messages to `Progman` or `WorkerW` using blocking `SendMessageW` can freeze KinetiWall if Windows Explorer is busy or hung.
* **Decision:** All window messages dispatched to shell windows must use `SendMessageTimeoutW` with `SMTO_NORMAL` and a maximum timeout of $1000\text{ ms}$.
* **Consequences:** Eliminates deadlock hazards. Core maintains supervisor responsiveness even if the desktop shell experiences a hard freeze.

### `DEC-DSK-03`: Enforce `WS_EX_TRANSPARENT | WS_EX_NOACTIVATE` on Video Players
* **Context:** Embedding a window across process boundaries via `SetParent` can synchronize thread input queues and steal mouse focus from desktop icons.
* **Decision:** All video player render windows must possess extended styles `WS_EX_TRANSPARENT | WS_EX_NOACTIVATE | WS_EX_LAYERED`.
* **Consequences:** All mouse clicks, double-clicks, and drag selections pass through completely to `SysListView32`. Zero keyboard focus stealing.

### `DEC-DSK-04`: Dual-Key Window Handle Validation
* **Context:** HWNDs can be invalidated or recycled if Explorer restarts during operation.
* **Decision:** Before manipulating any host window handle, KinetiWall must verify: (1) `IsWindow(hWorkerW)`, (2) `RealGetWindowClassW == "WorkerW"`, and (3) `GetWindowThreadProcessId` matches active Explorer PID.
* **Consequences:** Prevents operations on stale, recycled, or unrelated third-party window handles.

### `DEC-DSK-05`: Strict Per-Monitor DPI & Virtual Coordinate Normalization
* **Context:** Modern Windows systems frequently run mixed-DPI displays (e.g. 4K 150% + 1080p 100%) and negative virtual desktop coordinates.
* **Decision:** Declare `DPI_AWARENESS_CONTEXT_PER_MONITOR_AWARE_V2` on all player binaries. Calculate child window bounds relative to `SM_XVIRTUALSCREEN` and `SM_YVIRTUALSCREEN`.
* **Consequences:** Guarantees crisp physical pixel rendering without blur, scaling distortion, or clipping across multi-monitor layouts.

### `DEC-DSK-06`: Explicit Detach and Desktop Invalidation on Teardown
* **Context:** Terminating a player without unparenting can leave a black desktop surface or visual artifacts.
* **Decision:** During shutdown or wallpaper stop, the player must call `ShowWindow(SW_HIDE)`, `SetParent(NULL)`, and trigger `RedrawWindow` on `WorkerW` before destroying its own window.
* **Consequences:** Instantly restores the user's native Windows static desktop wallpaper seamlessly without shell flicker.

### `DEC-DSK-07`: Event-Driven Explorer Reconnection with Circuit Breaker
* **Context:** Polling for Explorer consumes unnecessary CPU, while an Explorer crash loop could cause continuous reconnection storms.
* **Decision:** Reconnect exclusively via `RegisterWindowMessageW("TaskbarCreated")`. Implement a circuit breaker limiting retries to 3 attempts with exponential backoff.
* **Consequences:** Keeps idle CPU usage $\le 0.05\%$. Enters a safe degraded state if Explorer experiences cascading crash failures.

### `DEC-DSK-08`: Quiescent Rendering Fallback for Occluded Displays
* **Context:** Rendering 60 FPS video when a fullscreen game or application covers the display wastes GPU/CPU cycles and drains battery.
* **Decision:** Detect fullscreen foreground windows via `GetForegroundWindow` and `GetMonitorInfoW`. When occluded, signal players to pause swapchain presentation and video decoding.
* **Consequences:** Reduces wallpaper background overhead to $0.0\%$ GPU and $< 0.1\%$ CPU during gaming or productivity, satisfying `KW-F-008`.

---

## 3. Open Engineering Review Items

| Item ID | Description | Current Architectural Status | Target Phase |
| :---: | :--- | :--- | :---: |
| **`REV-DSK-01`** | **Windows Copilot / Shell Sidebar Docking:** Evaluate whether Windows 11 Copilot docking dynamically resizes the `WorkerW` virtual desktop bounds. | Handled via `WM_DISPLAYCHANGE` debounce; requires validation on 23H2/24H2. | Phase 4 (Runtime Verification) |
| **`REV-DSK-02`** | **Interactive Web Wallpaper Input Routing:** Specify exact mouse hit-testing pipeline for web wallpapers that require user interaction without blocking icon selection. | Web wallpapers default to click-through; interactive toggle mode to be refined in Step 17. | Phase 4 (Runtime Verification) |
| **`REV-DSK-03`** | **Multiple Display HDR Color Management:** Evaluate DirectX swapchain color space conversion (`DXGI_COLOR_SPACE_RGB_FULL_G2084_NONE_P2020`) when one display is HDR and another is SDR. | Evaluated under D3D11 architecture; color space is initialized per display player. | Phase 4 (Runtime Verification) |

