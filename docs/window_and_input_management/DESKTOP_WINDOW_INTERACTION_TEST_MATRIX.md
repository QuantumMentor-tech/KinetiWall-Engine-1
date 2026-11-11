# Desktop Window Interaction & Input Test Matrix
**Document ID:** `KW-ZIN-004`  
**Deliverable:** D (Window Interaction Test Matrix)  
**Parent Document:** [README.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/window_and_input_management/README.md)  
**Classification:** Quality Assurance & System Verification Specification  

---

## 1. Test Strategy & Acceptance Framework

Testing window z-order, focus, and input routing requires verifying real observable user interactions on the Windows desktop shell. Testing is divided into four rigorous tiers:
1. **Tier 1: Style & Window Attribute Unit Tests** (Validates `dwStyle`, `dwExStyle`, coordinate clamping, and input flag transitions).
2. **Tier 2: Synthetic Win32 Input Tests** (Uses `SendInput` to inject synthetic mouse clicks and marquee drags on the desktop; asserts hit targets).
3. **Tier 3: Focus & Activation Audits** (Monitors `GetForegroundWindow()` and `GetFocus()` across startup, wallpaper switching, and UI cycles).
4. **Tier 4: Adversarial Stress & Chaos Scenarios** (Explorer kills, rapid UI open/close toggles, full-screen gaming transitions).

> [!IMPORTANT]
> All test cases below represent approved verification specifications. Their status is marked as `NOT RUN (DESIGN PHASE)` pending runtime implementation in Phase 4.

---

## 2. Acceptance Criteria Verification Matrix (`WIN-001` through `WIN-012`)

| Test ID | Test Title & Scope | Target Subsystems | Observable Acceptance Criteria | Status |
| :--- | :--- | :--- | :--- | :--- |
| **`WIN-001`** | **Desktop Layer Placement** | `WorkerW` & SwapChain | Live video wallpaper renders visually behind `SysListView32` desktop icons on all active displays. | `NOT RUN (DESIGN PHASE)` |
| **`WIN-002`** | **Zero Focus Stealing on Launch** | Process Initialization | Launching KinetiWall or switching wallpapers while user types in Notepad does not interrupt typing or steal focus. | `NOT RUN (DESIGN PHASE)` |
| **`WIN-003`** | **Desktop Icon Selection & Click** | Shell Input / Hit-Test | Single-click selects an icon; double-click launches desktop shortcuts with zero delay or mouse lag. | `NOT RUN (DESIGN PHASE)` |
| **`WIN-004`** | **Marquee Rubber-Band Drag** | Desktop Selection Input| Clicking empty desktop space and dragging a marquee box draws the selection rectangle smoothly without video tearing. | `NOT RUN (DESIGN PHASE)` |
| **`WIN-005`** | **Desktop Shell Context Menu** | Explorer Shell Context | Right-clicking on wallpaper surfaces opens the native Windows desktop context menu (`View`, `Sort by`, `New`). | `NOT RUN (DESIGN PHASE)` |
| **`WIN-006`** | **Taskbar & Start Menu Superiority**| Shell Z-Order (`Shell_TrayWnd`)| Taskbar, Start menu, Action Center, and notification toasts remain strictly on top of wallpaper across all monitors. | `NOT RUN (DESIGN PHASE)` |
| **`WIN-007`** | **UI Independence & 0 MB Exit** | `kinetiwall-ui.exe` | Closing the UI window via "X" terminates UI completely ($0.0\text{ MB}$ RAM); wallpaper playback continues uninterrupted. | `NOT RUN (DESIGN PHASE)` |
| **`WIN-008`** | **Reopening UI Zero Duplicate HWNDs**| UI & Player Lifecycle | Reopening settings UI from system tray opens the GUI window without creating duplicate wallpaper windows or resetting z-order. | `NOT RUN (DESIGN PHASE)` |
| **`WIN-009`** | **Web Interactive Mode Toggle** | WebView2 Input Router | Triggering interactive hotkey (`Win+Alt+I`) removes `WS_EX_TRANSPARENT`; mouse clicks interact with web canvas smoothly. | `NOT RUN (DESIGN PHASE)` |
| **`WIN-010`** | **Web Interactive Auto-Revert** | Inactivity Timeout Engine | Inactivity timer (30s) or pressing `Escape` automatically restores `WS_EX_TRANSPARENT`, re-enabling desktop icon clicks. | `NOT RUN (DESIGN PHASE)` |
| **`WIN-011`** | **Fullscreen Gaming Occlusion** | Hardware Occlusion Monitor| Launching a fullscreen DX12/Vulkan game pauses wallpaper swapchains, dropping CPU/GPU overhead to $\le 0.1\%$. | `NOT RUN (DESIGN PHASE)` |
| **`WIN-012`** | **Explorer Crash Recovery** | Shell Lifecycle Hook | Killing `explorer.exe` detaches player HWND; upon shell restart, player re-anchors to new `WorkerW` in $\le 500\text{ ms}$. | `NOT RUN (DESIGN PHASE)` |

---

## 3. Synthetic Input Test Procedures

### 3.1 Test `WIN-004`: Marquee Selection Rectangle Drag Test

* **Objective:** Prove that `WS_EX_TRANSPARENT` allows mouse drag messages (`WM_LBUTTONDOWN`, `WM_MOUSEMOVE`, `WM_LBUTTONUP`) to pass through to `SysListView32`.
* **Procedure:**
  1. Set active wallpaper on primary monitor ($1920 \times 1080$).
  2. Synthesize mouse press at $(500, 500)$ via `SendInput`.
  3. Synthesize mouse move to $(800, 800)$ over $100\text{ ms}$.
  4. Query Windows desktop shell: verify that a marquee selection rectangle is actively rendered by Explorer.
  5. Synthesize mouse release at $(800, 800)$.
* **Pass Condition:** Explorer successfully selects desktop icons inside the $[500, 500, 800, 800]$ bounding box; video wallpaper experiences zero dropped frames.

### 3.2 Test `WIN-002`: Zero Focus Stealing Verification

* **Objective:** Verify that wallpaper window initialization respects the active user's typing focus.
* **Procedure:**
  1. Open `notepad.exe` and bring it to foreground. Verify `GetForegroundWindow() == hNotepad`.
  2. Send IPC command to Core: `CMD_SET_WALLPAPER(Display1, new_video_wallpaper)`.
  3. Continuously sample `GetForegroundWindow()` at $5\text{ ms}$ intervals during player spawn and WorkerW reparenting.
* **Pass Condition:** `GetForegroundWindow()` remains equal to `hNotepad` at 100% of sample intervals. Focus is never stolen.

