# Win32 Window Styles & Native Behavior Reference
**Document ID:** `KW-ZIN-003`  
**Deliverable:** C (Window Styles and Native Behavior Reference)  
**Parent Document:** [README.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/window_and_input_management/README.md)  
**Classification:** Reference Specification / Win32 Window Management Standard  

---

## 1. Verified Native Window Styles

This document specifies the exact Win32 window styles (`dwStyle`) and extended styles (`dwExStyle`) approved for each window type in the KinetiWall architecture:

| Window Type | Approved `dwStyle` | Approved `dwExStyle` | Technical Rationale & Platform Behavior |
| :--- | :--- | :--- | :--- |
| **`VideoPlayer_Render`** | `WS_CHILD \| WS_VISIBLE \| WS_CLIPSIBLINGS` | `WS_EX_TRANSPARENT \| WS_EX_NOACTIVATE \| WS_EX_LAYERED` | **Child of WorkerW**: `WS_CHILD` binds to `WorkerW`. `WS_CLIPSIBLINGS` prevents overdrawing desktop icons. `WS_EX_TRANSPARENT` passes hit-testing directly to `SysListView32`. `WS_EX_NOACTIVATE` guarantees zero focus stealing. |
| **`WebPlayer_Render (Passive)`** | `WS_CHILD \| WS_VISIBLE \| WS_CLIPSIBLINGS` | `WS_EX_TRANSPARENT \| WS_EX_NOACTIVATE` | **Default Web Mode**: Bypasses Chromium hit-testing, allowing full interaction with desktop icons beneath web canvas. |
| **`WebPlayer_Render (Interactive)`**| `WS_CHILD \| WS_VISIBLE \| WS_CLIPSIBLINGS` | `WS_EX_NOACTIVATE` | **Interactive Mode**: `WS_EX_TRANSPARENT` removed dynamically via `SetWindowLongPtrW`, routing mouse clicks to WebView2 DOM without activating the window as foreground. |
| **`Core_HiddenHost`** | `WS_POPUP` | `0` | **Message Loop Anchor**: Hidden window ($0 \times 0$) for receiving `WM_DISPLAYCHANGE`, power broadcasts, and tray messages. |
| **`Settings_UI_Main`** | `WS_OVERLAPPEDWINDOW` | `WS_EX_APPWINDOW` | **Normal Top-Level App**: Standard interactive window with minimize/maximize/close controls, visible in Alt+Tab and Taskbar. |

---

## 2. In-Depth Analysis of Critical Extended Styles

### 2.1 `WS_EX_TRANSPARENT` (0x00000020)
* **Documented Win32 Behavior:** Specifies that a window should not be painted until siblings beneath the window (that were created by the same thread) have been painted. Furthermore, during hit-testing (`WM_NCHITTEST`), Windows ignores windows with this style and passes the test to the window beneath it.
* **KinetiWall Application:** Guarantees that mouse clicks, dragging, and hover effects pass directly through the video wallpaper surface to Windows Explorer's desktop icon listview (`SysListView32`).

### 2.2 `WS_EX_NOACTIVATE` (0x08000000)
* **Documented Win32 Behavior:** A top-level or child window with this style cannot be activated by user clicking. The system does not bring this window to the foreground when the user clicks it.
* **KinetiWall Application:** Prevents the wallpaper window from ever stealing active foreground status from user applications when clicked.

### 2.3 `WS_CLIPSIBLINGS` (0x04000000)
* **Documented Win32 Behavior:** Clips child windows relative to each other; that is, when a particular child window receives a `WM_PAINT` message, the `WS_CLIPSIBLINGS` style clips all other overlapping child windows out of the region of the child window to be updated.
* **KinetiWall Application:** Prevents the player window from painting over sibling desktop windows, ensuring desktop icon rendering remains crisp and uncorrupted.

---

## 3. Dynamic Style Switching Protocol (Interactive Mode Toggle)

Switching the web player between Passive Click-Through and Interactive Mode requires modifying extended window styles dynamically without destroying the window or resetting the WebView2 engine:

```rust
pub unsafe fn set_interactive_input_mode(h_wnd: HWND, enable_interactive: bool) -> Result<(), PlatformError> {
    if !IsWindow(h_wnd).as_bool() {
        return Err(PlatformError::HandleInvalidated("Player window invalid"));
    }

    let current_ex_style = GetWindowLongPtrW(h_wnd, GWL_EXSTYLE);
    let new_ex_style = if enable_interactive {
        // Remove WS_EX_TRANSPARENT to allow WebView2 to receive mouse input
        current_ex_style & !(WS_EX_TRANSPARENT.0 as isize)
    } else {
        // Add WS_EX_TRANSPARENT to restore click-through to desktop icons
        current_ex_style | (WS_EX_TRANSPARENT.0 as isize)
    };

    if new_ex_style != current_ex_style {
        SetWindowLongPtrW(h_wnd, GWL_EXSTYLE, new_ex_style);
        
        // Force Windows window manager to recognize extended style change immediately
        SetWindowPos(
            h_wnd,
            HWND_TOP,
            0, 0, 0, 0,
            SWP_NOMOVE | SWP_NOSIZE | SWP_NOZORDER | SWP_NOACTIVATE | SWP_FRAMECHANGED,
        );
    }

    Ok(())
}
```

* **`SWP_FRAMECHANGED` Invariant:** Windows caches extended styles internally. Calling `SetWindowPos` with `SWP_FRAMECHANGED` instructs User32 to flush its hit-testing cache immediately without moving or resizing the window.

