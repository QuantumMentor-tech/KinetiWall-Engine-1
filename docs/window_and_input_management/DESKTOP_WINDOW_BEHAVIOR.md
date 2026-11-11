# Desktop Window Behavior Specification
**Document ID:** `KW-ZIN-001`  
**Deliverable:** A (Desktop Window Behavior Specification)  
**Parent Document:** [README.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/window_and_input_management/README.md)  
**Classification:** Desktop Windowing, Input Architecture & Shell Usability Standard  

---

## 1. Z-Order Hierarchy & Visual Stacking Model

The Windows Desktop composition engine operates as a strictly layered hierarchy. KinetiWall enforces a deterministic visual stacking order:

```
+---------------------------------------------------------------------------------+
| TOP LEVEL (Z-Top): System Toasts, Action Center, Lock Screen Overlays           |
+---------------------------------------------------------------------------------+
| SHELL CONTROLS: Windows Taskbar (Shell_TrayWnd), Start Menu, System Tray        |
+---------------------------------------------------------------------------------+
| FOREGROUND APPS: Active Fullscreen & Windowed Applications (IDEs, Games, Apps)  |
+---------------------------------------------------------------------------------+
| DESKTOP ICONS: SysListView32 (Inside WorkerW [Icon Host] / Progman)             |
|   - "Recycle Bin", User Files, Desktop Shortcuts, Marquee Selection Rectangle   |
+---------------------------------------------------------------------------------+
| === DESKTOP INTEGRATION LAYER (KinetiWall Target) ===                           |
|                                                                                 |
|   WorkerW [Wallpaper Host] (Created via Progman 0x052C message)                 |
|     |                                                                           |
|     +--- Child: KinetiWall Render Window (KinetiWall_VideoPlayer_Render)        |
|            - Styles: WS_CHILD | WS_VISIBLE | WS_CLIPSIBLINGS                    |
|            - ExtStyles: WS_EX_TRANSPARENT | WS_EX_NOACTIVATE | WS_EX_LAYERED    |
|            - SwapChain: Direct3D 11 / DXGI                                      |
+---------------------------------------------------------------------------------+
| BACKGROUND BASE: Native Shell Static Desktop Wallpaper Bitmap                   |
+---------------------------------------------------------------------------------+
```

### 1.1 Inviolable Z-Order Invariants
1. **Desktop Icons Remain Topmost:** Under no operational condition may a KinetiWall window elevate above `SysListView32`. Desktop icons must always remain 100% visible and interactive.
2. **Taskbar Superiority:** The Windows Taskbar (`Shell_TrayWnd`) and any secondary monitor taskbars (`Shell_SecondaryTrayWnd`) must never be obscured, clipped, or overlaid by a wallpaper window.
3. **No Elevation on Minimize:** When the user presses `Win+D` (Show Desktop), Windows minimizes top-level application windows. Because KinetiWall's renderers are embedded as child windows inside `WorkerW`, they are immune to minimization and remain visible behind icons.

---

## 2. Parenting, Ownership, and Activation Semantics

Windows Win32 architecture distinguishes between three related but distinct window concepts:

| Concept | Win32 Representation | KinetiWall Architectural Application | Security & Concurrency Implications |
| :--- | :--- | :--- | :--- |
| **Parenting** | `SetParent(hChild, hParent)` | Player render HWND is reparented into the shell's `WorkerW` host. | Changes coordinate space from virtual screen to WorkerW client space. Does *not* transfer process ownership. |
| **Ownership** | `GetWindow(hWnd, GW_OWNER)` | Top-level windows have owner HWNDs (e.g. dialogs owned by UI). | KinetiWall player windows have **no owner window** (`GW_OWNER == NULL`), preventing dialog cascade bugs. |
| **Activation** | `SetForegroundWindow(hWnd)` | Active window receiving keyboard input focus. | Wallpaper windows **must never be activated**. All windowing operations must specify `SWP_NOACTIVATE`. |

---

## 3. Window Activation & Focus Prevention Policy

Live wallpaper windows must operate as "ghost" surfaces from the perspective of Windows window management:

```
[Window Operation Trigger (Create / Resize / Move / Resume)]
                            |
                            v
   Execute Win32 Positioning via SetWindowPos:
   - HWND_TOP (Within WorkerW Child Hierarchy)
   - Flags: SWP_NOACTIVATE | SWP_NOZORDER | SWP_SHOWWINDOW
                            |
                            v
   Post-Operation Verification:
   Assert: GetForegroundWindow() != hPlayerWnd
   Assert: GetFocus() != hPlayerWnd
```

### 3.1 Strict Focus Prevention Invariants
1. **Creation Without Activation:** Window creation must never specify `WS_VISIBLE` in `CreateWindowExW`. The window is created hidden, configured, and then made visible strictly using `ShowWindow(hWnd, SW_SHOWNOACTIVATE)`.
2. **Foreground Window Preservation:** During display hotplugging or resolution changes, the currently active foreground window (`GetForegroundWindow()`) must not lose focus or experience a focus flicker.
3. **No Global Keystroke Interception:** Wallpaper windows must never register global hotkeys or call `GetAsyncKeyState` outside of authorized in-app configuration shortcuts (`DEC-SEC-01`).

---

## 4. Anti-Thrashing Z-Order Reconciliation Policy

A common bug in wallpaper software is running a background timer that continuously calls `SetWindowPos(HWND_BOTTOM)` every 100 ms to enforce placement. This causes severe GPU compositor thrashing, desktop flicker, and elevated CPU usage ($\ge 5\%$).

### 4.1 Event-Driven Reconciliation Rules
KinetiWall strictly forbids periodic z-order polling (`Principle H`):
* **Allowed Re-Anchoring Triggers:**
  1. *Host Invalidation:* `IsWindow(hWorkerW) == FALSE` (e.g. Explorer restart).
  2. *Display Topology Changes:* Validated `WM_DISPLAYCHANGE` notification after 500 ms debounce.
  3. *Shell Restart:* Receipt of registered `TaskbarCreated` message.
* **Forbidden Triggers:**
  - Timer-based polling loops.
  - Foreground application switch events (`EVENT_SYSTEM_FOREGROUND`).
  - Mouse movement or desktop click events.

---

## 5. UI and Desktop Shell Coexistence

The KinetiWall configuration and gallery GUI (`kinetiwall-ui.exe`) is completely decoupled from active wallpaper playback:

```
[Action: User Opens Settings UI]
  --> kinetiwall-ui.exe launches as a standard top-level desktop window.
  --> Acquires foreground focus normally; renders library and controls.
  --> Active wallpaper players continue rendering at 60 FPS in WorkerW without interruption.

[Action: User Closes Settings UI]
  --> kinetiwall-ui.exe intercepts WM_CLOSE.
  --> Flushes uncommitted settings to Core via IPC.
  --> Calls ExitProcess(0). RAM drops to 0.0 MB.
  --> Wallpaper playback continues uninterrupted; ZERO z-order changes occur.
```

---

## 6. Fullscreen Application Coordination & Occlusion Management

To comply with `KW-F-008` (power saving and zero resource waste during gaming or productivity):
1. **Passive Occlusion Query:** Core monitors foreground window transitions via `SetWinEventHook(EVENT_SYSTEM_FOREGROUND)`.
2. **Fullscreen Detection:** If the foreground window's bounding rect matches or exceeds the monitor rect (`GetMonitorInfoW`), and its window class is *not* `WorkerW` or `Progman`, the display is marked **OCCLUDED**.
3. **Quiescent Rendering Pause:** Core dispatches `CMD_PAUSE_DISPLAY(DisplayId)` to the player. The player skips swapchain `Present` calls and halts video decoding, reducing CPU/GPU overhead to $\le 0.1\%$ without modifying window z-order or detaching from `WorkerW`.
4. **Instant Resume:** When the fullscreen window minimizes or closes, Core sends `CMD_RESUME_DISPLAY(DisplayId)` to instantly resume playback without recreating windows.

