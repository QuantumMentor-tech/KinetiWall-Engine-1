# Desktop Window Z-Order, Focus, Input Routing & Shell Interaction Specification
**Document ID:** `KW-ZIN-000`  
**Phase:** 4 (Windows Platform Integration & Desktop Hosting)  
**Step:** 19 of 90  
**Status:** Approved Architectural Specification  
**Classification:** Desktop Windowing, Input Architecture & Shell Usability Standard  

---

## 1. Document Overview and Scope

This document suite establishes the authoritative engineering specification for **Desktop Window Z-Order Layering, Window Activation, Keyboard and Pointer Focus, Input Routing, and Windows Shell Coexistence** for the **KinetiWall** live wallpaper engine.

Live wallpaper applications on Microsoft Windows must achieve visual immersion without disrupting the core usability of the operating system:
1. **Layer Hierarchy Separation:** Renderers must reside visually *beneath* desktop icons (`SysListView32`), *beneath* the Windows Taskbar (`Shell_TrayWnd`), and *beneath* all active foreground applications, while painting smoothly over the native shell wallpaper surface.
2. **Zero Focus Stealing:** Wallpaper window creation, resizing, reparenting, and resume events must never steal keyboard focus or activate as the foreground window, preserving the user's active typing context in IDEs, games, and productivity applications.
3. **Passive Click-Through Default:** Non-interactive video wallpapers (`kinetiwall-player-video.exe`) must pass all pointer and mouse interactions through to the desktop icon listview underneath without delay or interception.
4. **Controlled Interactive Web Input:** For interactive web wallpapers (`kinetiwall-player-web.exe`), input routing must support dual-mode operation: defaulting to transparent click-through, with an explicit user toggle mechanism to engage mouse interaction without capturing global keyboard shortcuts or breaking desktop context menus.
5. **Shell Usability Protection:** Standard Windows desktop capabilities—rubber-band marquee icon selection, double-clicking desktop shortcuts, right-clicking to open the Windows shell context menu, Taskbar auto-hide, and Start menu activation—must remain 100% operational.

This specification distinguishes HWND parenting from window ownership, formalizes input routing pipelines, defines window styles and extended styles, provides a 12-point acceptance matrix, and establishes an evidence-based risk register.

---

## 2. Core Desktop Interaction Principles (Principles A through L)

The window management, focus, and input routing architecture is strictly governed by twelve engineering principles:

* **Principle A: Distinction of Windowing Concepts**  
  Parenting (`SetParent`), window ownership (`GetWindow(GW_OWNER)`), visual z-order (`HWND_BOTTOM` / `WS_CLIPSIBLINGS`), window activation (`SetForegroundWindow`), and input focus (`SetFocus`) are distinct operating system mechanisms. A live wallpaper window must never conflate visual placement with activation authority.
* **Principle B: Zero Foreground Interference**  
  Wallpaper windows must never become the active foreground window (`GetForegroundWindow()`). Window positioning and visibility modifications must use `SWP_NOACTIVATE` and `SWP_NOZORDER` to eliminate focus stealing and typing interruption.
* **Principle C: Transparent Click-Through for Video Players**  
  All video wallpaper rendering windows (`kinetiwall-player-video.exe`) must possess extended styles `WS_EX_TRANSPARENT | WS_EX_NOACTIVATE | WS_EX_LAYERED`, ensuring mouse clicks, double-clicks, and drag events pass directly through to `SysListView32` without OS message queue delays.
* **Principle D: Dual-Mode Input for Interactive Web Wallpapers**  
  Interactive web wallpapers (`kinetiwall-player-web.exe`) default to passive click-through mode. An explicit, user-configured interaction trigger (e.g. desktop hotkey or tray toggle) enables web input routing while strictly preserving desktop context menus and global keyboard shortcuts.
* **Principle E: Prohibition of Covert Global Input Hooks**  
  KinetiWall must never install global low-level keyboard hooks (`WH_KEYBOARD_LL`) or global mouse hooks (`WH_MOUSE_LL`) to intercept or snoop user input (`DEC-SEC-01`). Input handling is strictly scoped to application-owned windows.
* **Principle F: Unobstructed Shell Desktop Usability**  
  Desktop icon marquee selection (rubber-band selection rectangle), file drag-and-drop, desktop right-click context menus (`View`, `Sort by`, `New`), and Desktop Spotlight controls must function with zero visual tearing or event swallowing.
* **Principle G: Absolute Taskbar & Start Menu Invariance**  
  The Windows Taskbar (`Shell_TrayWnd`), secondary taskbars (`Shell_SecondaryTrayWnd`), Start menu (`Windows.UI.Core.CoreWindow`), and Action Center notification toasts must always maintain z-order superiority over all wallpaper windows.
* **Principle H: Anti-Thrashing Z-Order Reconciliation**  
  KinetiWall must never run aggressive polling loops or continuous `SetWindowPos(HWND_BOTTOM)` enforcement. Re-anchoring is strictly event-driven, triggered only by validated shell events (`TaskbarCreated`, `WM_DISPLAYCHANGE`, host invalidation).
* **Principle I: Ephemeral UI Focus Independence**  
  Opening, interacting with, or closing the settings GUI (`kinetiwall-ui.exe`) must never alter the z-order, visibility, or rendering state of active wallpaper windows. When closed, `kinetiwall-ui.exe` terminates completely ($0.0\text{ MB}$ footprint) without impacting renderers (`KW-NG-001`).
* **Principle J: Fullscreen Application Respect & Occlusion Pause**  
  When a foreground application covers an active display in fullscreen (games, video players, presentations), Core marks the display occluded and halts swapchain presentation, dropping CPU/GPU usage to $\le 0.1\%$ without modifying window z-order.
* **Principle K: Safe Quiescent Fallback on Host Loss**  
  If the `WorkerW` host is invalidated or destroyed (e.g. Explorer crash), the player window immediately detaches and hides itself (`ShowWindow(SW_HIDE)`), restoring native Windows desktop wallpaper rather than promoting itself to a floating top-level window.
* **Principle L: Traceable & Redacted Diagnostic Logging**  
  Window creation, style modifications, activation anomalies, and input mode transitions emit structured diagnostic events. Raw user keystrokes, mouse coordinates, and third-party window titles are strictly redacted.

---

## 3. Documentation Suite Index

| Deliverable | File Path | Focus & Architectural Scope |
| :--- | :--- | :--- |
| **Index & Principles** | [README.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/window_and_input_management/README.md) | Architectural index, 12 core interaction principles, requirements traceability matrix. |
| **Deliverable A** | [DESKTOP_WINDOW_BEHAVIOR.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/window_and_input_management/DESKTOP_WINDOW_BEHAVIOR.md) | Z-order hierarchy, parenting vs ownership, activation rules, anti-thrashing reconciliation, fullscreen coordination, and UI coexistence. |
| **Deliverable B** | [DESKTOP_INPUT_AND_FOCUS_POLICY.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/window_and_input_management/DESKTOP_INPUT_AND_FOCUS_POLICY.md) | Passive video click-through pipeline, interactive web dual-mode input, mouse routing, keyboard focus isolation, and accessibility. |
| **Deliverable C** | [WIN32_WINDOW_BEHAVIOR_REFERENCE.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/window_and_input_management/WIN32_WINDOW_BEHAVIOR_REFERENCE.md) | Verified Win32 window styles (`WS_CHILD`, `WS_CLIPSIBLINGS`), extended styles (`WS_EX_TRANSPARENT`, `WS_EX_NOACTIVATE`), and API semantics. |
| **Deliverable D** | [DESKTOP_WINDOW_INTERACTION_TEST_MATRIX.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/window_and_input_management/DESKTOP_WINDOW_INTERACTION_TEST_MATRIX.md) | 12-point acceptance matrix (`WIN-001` through `WIN-012`), synthetic input testing, marquee drag tests, and automated CI test harness. |
| **Deliverable E** | [WINDOW_INTERACTION_RISK_REGISTER.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/window_and_input_management/WINDOW_INTERACTION_RISK_REGISTER.md) | Risk register (`RSK-ZIN-01..08`), Architecture Decision Records (`DEC-ZIN-01..08`), and open engineering review items. |

---

## 4. Requirements Traceability Matrix

| Requirement / Decision | Description | Governing Specification Section |
| :--- | :--- | :--- |
| **`KW-F-001`** | Core engine initialization and native desktop shell integration | `DESKTOP_WINDOW_BEHAVIOR.md` §2 |
| **`KW-F-002`** | High-performance video wallpaper playback engine (mpv / D3D11) | `DESKTOP_INPUT_AND_FOCUS_POLICY.md` §2 |
| **`KW-F-003`** | Interactive web wallpaper playback engine (WebView2) | `DESKTOP_INPUT_AND_FOCUS_POLICY.md` §3 |
| **`KW-F-008`** | Fullscreen hardware occlusion detection & power pause | `DESKTOP_WINDOW_BEHAVIOR.md` §6 |
| **`KW-NF-001`** | CPU utilization limits ($\le 1.0\%$ idle background overhead) | `DESKTOP_WINDOW_BEHAVIOR.md` §4 |
| **`KW-NF-006`** | Graceful shutdown and rapid exit response ($\le 250\text{ ms}$) | `DESKTOP_WINDOW_BEHAVIOR.md` §5 |
| **`KW-NG-001`** | UI ephemeral memory guarantee ($0\text{ MB}$ resident footprint) | `DESKTOP_WINDOW_BEHAVIOR.md` §5 |
| **`ADR-0002`** | Isolated multi-process topology (Core, UI, Per-Display Players) | `DESKTOP_WINDOW_BEHAVIOR.md` §3 |
| **`DEC-DSK-01`** | WorkerW shell integration approval via `0x052C` | `DESKTOP_WINDOW_BEHAVIOR.md` §2 |
| **`DEC-DSK-03`** | Enforce `WS_EX_TRANSPARENT | WS_EX_NOACTIVATE` on video players | `WIN32_WINDOW_BEHAVIOR_REFERENCE.md` §2 |
| **`DEC-ZIN-01..08`**| Dedicated windowing, focus, and input routing architecture decisions | `WINDOW_INTERACTION_RISK_REGISTER.md` §2 |

