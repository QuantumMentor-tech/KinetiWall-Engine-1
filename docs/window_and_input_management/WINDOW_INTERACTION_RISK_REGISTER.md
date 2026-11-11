# Desktop Window Interaction Risk Register & Decisions
**Document ID:** `KW-ZIN-005`  
**Deliverable:** E (Window Interaction Risk Register)  
**Parent Document:** [README.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/window_and_input_management/README.md)  
**Classification:** Core System Architecture / Operational Reliability Standard  

---

## 1. Desktop Window Interaction Risk Register

Embedding custom render surfaces directly into the Windows desktop shell introduces subtle user experience and operating system risks:

| Risk ID | Risk Summary & Failure Mode | Prob. | Impact | Severity | Architecture Mitigation & Prevention Strategy |
| :--- | :--- | :---: | :---: | :---: | :--- |
| **`RSK-ZIN-01`** | **Wallpaper Steals User Typing Focus**<br>During player spawn or resolution change, the wallpaper window briefly activates, interrupting user typing in games/apps. | High | High | **CRITICAL** | **Mandatory `SWP_NOACTIVATE` & `SW_SHOWNOACTIVATE` (`DEC-ZIN-01`)**: All windowing operations strictly omit activation flags; creation occurs hidden. |
| **`RSK-ZIN-02`** | **Desktop Icons Blocked by Player Window**<br>Video player reparented without `WS_EX_TRANSPARENT`, preventing user from selecting icons or right-clicking desktop. | Med | High | **CRITICAL** | **Enforced Extended Transparent Style (`DEC-ZIN-02`)**: Video windows must have `WS_EX_TRANSPARENT` set upon creation; User32 bypasses hit-testing. |
| **`RSK-ZIN-03`** | **Z-Order Thrashing Polling Loop**<br>Naive implementation polls `SetWindowPos(HWND_BOTTOM)` on a 100ms timer, causing continuous DWM compositor flicker and high CPU. | High | Med | **HIGH** | **Anti-Thrashing Event-Driven Model (`DEC-ZIN-03`)**: Zero periodic z-order enforcement. Re-anchoring occurs strictly on validated OS events (`WM_DISPLAYCHANGE`, `TaskbarCreated`). |
| **`RSK-ZIN-04`** | **Interactive Web Wallpaper Input Trap**<br>User engages web interaction mode; wallpaper traps all mouse clicks, permanently preventing desktop icon access. | Med | High | **HIGH** | **Inactivity Timeout & Emergency Escape (`DEC-ZIN-04`)**: 30-second mouse idle timer and `Escape` key automatically revert to passive click-through mode. |
| **`RSK-ZIN-05`** | **Taskbar Occlusion during Spanning**<br>Spanning wallpaper window bounds accidentally cover the Windows Taskbar, making system navigation impossible. | Low | High | **CRITICAL** | **Shell Z-Order Invariant & WorkerW Hierarchy (`DEC-ZIN-05`)**: Player is child of `WorkerW`, which is structurally positioned behind `Shell_TrayWnd` by the Windows kernel. |
| **`RSK-ZIN-06`** | **Thread Queue Deadlock via `SetParent`**<br>Calling `SetParent` across processes synchronizes thread input queues, deadlocking if Explorer hangs. | Med | High | **HIGH** | **Decoupled Input Queues via Transparent Styles (`DEC-ZIN-06`)**: `WS_EX_TRANSPARENT` prevents input routing to the child, eliminating thread queue synchronization overhead. |
| **`RSK-ZIN-07`** | **Ghost Window on Explorer Restart**<br>Explorer crashes; child window is orphaned or promoted to floating top-level window above all applications. | Med | High | **HIGH** | **Quiescent Detach & Hide on Host Loss (`DEC-ZIN-07`)**: Core detects Explorer loss, commands player to invoke `ShowWindow(SW_HIDE)` and detach immediately. |
| **`RSK-ZIN-08`** | **Global Input Interception Security Risk**<br>Using low-level hooks (`WH_KEYBOARD_LL`) to capture keystrokes triggers antivirus heuristics or introduces security vulnerabilities. | Low | High | **CRITICAL** | **Absolute Prohibition of Global Input Hooks (`DEC-ZIN-08`)**: Keystroke hooks are forbidden; interactive input uses local window message loops only. |

---

## 2. Architecture Decision Records (Window Interaction: `DEC-ZIN-01` to `08`)

The following formal decisions govern all window placement, focus, and input routing implementations across KinetiWall:

### `DEC-ZIN-01`: Absolute Focus Stealing Prevention via `SWP_NOACTIVATE`
* **Context:** Wallpaper initialization must never steal focus from active applications or games.
* **Decision:** All window creation and positioning calls must use `SW_SHOWNOACTIVATE` and `SWP_NOACTIVATE | SWP_NOZORDER`.
* **Consequences:** Completely prevents focus stealing. Users can type in active foreground applications while wallpapers switch seamlessly in the background.

### `DEC-ZIN-02`: Mandatory `WS_EX_TRANSPARENT` for All Video Players
* **Context:** Video wallpapers do not accept user input and must not obstruct desktop icons.
* **Decision:** Enforce `WS_EX_TRANSPARENT | WS_EX_NOACTIVATE | WS_EX_LAYERED` on `kinetiwall-player-video.exe` windows.
* **Consequences:** User32 hit-testing bypasses the video window entirely, passing all clicks, marquee drags, and right-click context menu events directly to `SysListView32`.

### `DEC-ZIN-03`: Prohibition of Periodic Z-Order Polling Loops
* **Context:** Calling `SetWindowPos` periodically causes GPU composition thrashing and high idle CPU consumption.
* **Decision:** Z-order adjustments are strictly prohibited outside of validated event triggers (`WM_DISPLAYCHANGE`, `TaskbarCreated`, cold start).
* **Consequences:** Keeps idle background CPU overhead $\le 0.05\%$, satisfying `KW-NF-001`.

### `DEC-ZIN-04`: Dual-Mode Input with Auto-Revert for Web Wallpapers
* **Context:** Interactive web wallpapers need mouse interaction, but must not permanently disable desktop icon access.
* **Decision:** Implement Dual-Mode Input: defaults to passive click-through (`WS_EX_TRANSPARENT`); interactive mode is toggled via hotkey/tray and automatically reverts after 30 seconds of inactivity or pressing `Escape`.
* **Consequences:** Provides rich HTML5/WebGL interaction while eliminating input traps.

### `DEC-ZIN-05`: Strict Child Placement within `WorkerW` Beneath Shell Controls
* **Context:** Wallpaper windows must never occlude the Windows Taskbar or Start menu.
* **Decision:** Renderers are embedded strictly as child windows of `WorkerW` using `WS_CHILD | WS_CLIPSIBLINGS`.
* **Consequences:** The Windows NT compositor guarantees that `Shell_TrayWnd` and the Start menu reside on higher visual planes, making taskbar occlusion physically impossible.

### `DEC-ZIN-06`: Cross-Process Message Queue Decoupling
* **Context:** Win32 `SetParent` can synchronize thread input queues across processes, risking deadlocks.
* **Decision:** Combining `SetParent` with `WS_EX_TRANSPARENT | WS_EX_NOACTIVATE` ensures Windows routes 0 input messages to the child window, preventing queue synchronization.
* **Consequences:** Decouples player render threads from Explorer's main message pump, eliminating hang propagation.

### `DEC-ZIN-07`: Quiescent Detachment and Hide on Shell Loss
* **Context:** When `explorer.exe` restarts, child windows embedded inside `WorkerW` can float over ordinary applications if not handled.
* **Decision:** Upon host invalidation, players immediately execute `ShowWindow(SW_HIDE)` and detach (`SetParent(NULL)`).
* **Consequences:** Restores native Windows static wallpaper during Explorer restarts; zero misplaced or orphaned floating windows.

### `DEC-ZIN-08`: Total Prohibition of Global Low-Level Input Hooks
* **Context:** Installing `WH_KEYBOARD_LL` or `WH_MOUSE_LL` hooks creates severe security vulnerabilities, triggers false-positive AV detections, and causes system latency.
* **Decision:** KinetiWall strictly forbids the creation or use of global input hooks across all components.
* **Consequences:** Enforces compliance with `DEC-SEC-01` and eliminates keylogger risks.

---

## 3. Open Engineering Review Items

| Item ID | Description | Current Architectural Status | Target Phase |
| :--- | :--- | :--- | :---: |
| **`REV-ZIN-01`** | **Desktop Spotlight Context Menu Collision:** Evaluate whether Windows 11 Desktop Spotlight desktop widget interacts cleanly with `WS_EX_TRANSPARENT` click-through. | Spotlight operates inside `SysListView32`; click-through confirmed in initial research; scheduled for verification. | Phase 4 (Runtime Verification) |
| **`REV-ZIN-02`** | **Touch & Pen Pointer Normalization:** Verify whether Windows Ink stylus gestures pass cleanly through `WS_EX_TRANSPARENT` to desktop icons. | Mouse input verified; touch/pen hit-testing to be validated on Surface hardware. | Phase 4 (Runtime Verification) |
| **`REV-ZIN-03`** | **WebView2 Hardware Context Menu:** Refine right-click behavior when web wallpapers operate in interactive mode (suppressing default browser context menu in favor of desktop menu). | Default context menu disabled in WebView2 settings (`AreDefaultContextMenusEnabled = false`). | Phase 4 (Runtime Verification) |

