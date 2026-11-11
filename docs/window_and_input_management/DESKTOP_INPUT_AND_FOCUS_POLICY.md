# Desktop Input Routing & Focus Policy
**Document ID:** `KW-ZIN-002`  
**Deliverable:** B (Input Routing and Focus Policy)  
**Parent Document:** [README.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/window_and_input_management/README.md)  
**Classification:** Desktop Windowing, Input Architecture & Shell Usability Standard  

---

## 1. Input Routing Architecture & Operational Models

Input handling in KinetiWall is strictly partitioned based on the active wallpaper engine:

```
                                  [USER MOUSE & POINTER INPUT]
                                               |
                                               v
                                [Windows Desktop Coordinate Space]
                                               |
               +-------------------------------+-------------------------------+
               |                                                               |
     [VIDEO WALLPAPER ENGINE]                                        [WEB WALLPAPER ENGINE]
  (kinetiwall-player-video.exe)                                    (kinetiwall-player-web.exe)
               |                                                               |
               v                                                               v
    [WS_EX_TRANSPARENT Active]                                      [DUAL-MODE INPUT ROUTER]
  Windows Kernel Bypasses Window                                               |
               |                                       +-----------------------+-----------------------+
               v                                       |                                               |
   [PASS DIRECTLY TO SHELL]                   [PASSIVE MODE (Default)]                     [INTERACTIVE MODE]
               |                               WS_EX_TRANSPARENT Active                   WS_EX_TRANSPARENT Removed
               v                                       |                                               |
   - SysListView32 Desktop Icons                       v                                               v
   - Marquee Rubber-Band Selection         Input Passes Directly to Icons               Mouse Routed to WebView2 Engine
   - Shell Desktop Context Menu                                                         (HTML5 / Canvas / WebGL Clicks)
```

---

## 2. Passive Video Wallpaper Input Pipeline (`kinetiwall-player-video.exe`)

Non-interactive video wallpapers (MP4, MKV, WebM, animated GIFs) require zero user input. They must exist as purely visual presentation surfaces:

1. **Window Style Invariant:** The video render window must be configured with extended styles:
   $$\text{dwExStyle} = \text{WS\_EX\_TRANSPARENT} \mid \text{WS\_EX\_NOACTIVATE} \mid \text{WS\_EX\_LAYERED}$$
2. **Win32 Hit-Testing Semantics:** Under `WS_EX_TRANSPARENT`, the Windows window manager completely ignores the window during `WM_NCHITTEST` queries. Hit-testing immediately falls through to the sibling `WorkerW` hosting `SHELLDLL_DefView`.
3. **Guaranteed Shell Functionality:**
   * **Single-Click / Double-Click:** Clicks on desktop icons launch shortcuts with zero latency.
   * **Marquee Selection:** Clicking and dragging on empty desktop space draws the native Windows translucent selection rectangle smoothly without video frame drops.
   * **Shell Context Menu:** Right-clicking any wallpaper area instantly opens the Windows desktop context menu (`View`, `Sort by`, `Personalize`).

---

## 3. Interactive Web Wallpaper Input Pipeline (`kinetiwall-player-web.exe`)

Interactive web wallpapers (HTML5/Canvas, Three.js, WebGL, particle simulations) present a delicate architectural challenge: allowing mouse interaction with web elements without breaking desktop icon usability.

### 3.1 Dual-Mode Input Architecture

To solve this conflict, KinetiWall implements a **Dual-Mode Input Policy**:

| Input Mode | Activation Trigger | Extended Window Styles | Mouse Routing Destination | Shell Icon Usability | Context Menu Usability |
| :--- | :--- | :--- | :--- | :---: | :---: |
| **Mode 1: Passive Click-Through (Default)** | Default on wallpaper load; user returns to productivity. | `WS_EX_TRANSPARENT \| WS_EX_NOACTIVATE` | Bypasses WebView2; passes directly to `SysListView32`. | **100% Intact** (Icons fully selectable) | **100% Intact** (Shell context menu opens) |
| **Mode 2: Interactive Web Mode** | User clicks tray "Enable Interaction" or presses global toggle hotkey. | `WS_EX_NOACTIVATE` (`WS_EX_TRANSPARENT` removed) | Mouse clicks & moves routed directly to WebView2 DOM. | Suspended (Clicks interact with web canvas) | Web Context Menu (or custom script menu) |

### 3.2 Transition Mechanics & Anti-Trap Safeguards

To prevent a rogue web wallpaper from trapping user input or blocking desktop access:
1. **Explicit Toggle Ingress:** The user toggles interactive mode via a customizable hotkey (default: `Win + Alt + I`) or via the notification tray icon.
2. **Auto-Revert Inactivity Timeout:** If interactive mode is enabled, KinetiWall starts a 30-second idle timer. If no mouse movement occurs for 30 seconds, KinetiWall automatically reverts the window to Passive Click-Through mode (`WS_EX_TRANSPARENT`), restoring desktop icon selection.
3. **Emergency Escape:** Pressing `Escape` or switching to any foreground application immediately disengages Interactive Mode.

---

## 4. Keyboard Focus & Accessibility Isolation

1. **Strict Keyboard Rejection:** Live wallpaper windows **never accept keyboard focus** (`SetFocus`). All `WM_KEYDOWN`, `WM_KEYUP`, and `WM_CHAR` messages directed to the desktop are handled by Windows Explorer for desktop icon search navigation (typing letters selects matching desktop filenames).
2. **Prohibition of Global Keystroke Hooks:** In strict accordance with `DEC-SEC-01`, KinetiWall does not install `WH_KEYBOARD_LL` hooks. Interactive web wallpapers cannot capture arbitrary user keystrokes, completely preventing keylogger vulnerabilities.
3. **Accessibility Compatibility:** Windows Narrator, UI Automation, and high-contrast accessibility tools query `SysListView32` without interference from KinetiWall's transparent rendering surface.

