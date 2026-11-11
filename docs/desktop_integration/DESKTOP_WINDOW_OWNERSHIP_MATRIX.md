# Desktop Window Ownership & Resource Matrix
**Document ID:** `KW-DSK-003`  
**Deliverable:** C (Native Window Ownership and Resource Matrix)  
**Parent Document:** [README.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/desktop_integration/README.md)  
**Classification:** System Architecture Reference / Resource Ownership Standard  

---

## 1. Window Ownership Architecture

Windows OS enforces strict ownership and message dispatch rules across threads and processes. A critical hazard in desktop integration is inadvertently attaching thread message queues via `SetParent` or creating windows with conflicting styles.

### 1.1 Process & Window Boundaries

```
[PROCESS: explorer.exe]
  |
  +-- Progman (Top-Level)
  +-- WorkerW [Icon Host] ---> Child: SHELLDLL_DefView ---> Child: SysListView32
  +-- WorkerW [Target Host] ---------------------------------------------+
                                                                         | (Cross-Process SetParent)
[PROCESS: kinetiwall-player-video.exe (Display 1)]                       |
  |                                                                      |
  +-- Render Window HWND (class "KinetiWall_VideoPlayer_Render") <-------+
        - Created by Player Process Thread
        - Owned exclusively by Player Process
        - Subclassed as child of WorkerW
        - SwapChain Surface: Direct3D 11

[PROCESS: kinetiwall-core.exe]
  |
  +-- Hidden Message Window HWND (class "KinetiWall_Core_MessageLoop")
        - Receives WM_DISPLAYCHANGE, PBT_APMPOWERBROADCAST, TaskbarCreated
        - Notification Tray Icon Anchor
```

---

## 2. Comprehensive Desktop Window Ownership Matrix

| Window Category | Owning Process | Owning Thread | Window Class | Parent / Host Relationship | Mandatory Window Styles | Target Display Geometry | Input & Focus Behavior | Destroying Component | Cleanup Signal / Verification |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **`WIN-01: VideoPlayerRender`** | `kinetiwall-player-video` | Player Render Thread | `KinetiWall_VideoPlayer_Render` | Child of `WorkerW` (`SetParent`) | `WS_CHILD \| WS_VISIBLE \| WS_CLIPSIBLINGS` / `WS_EX_TRANSPARENT \| WS_EX_NOACTIVATE` | Bound to Display 1 physical rect (`GetMonitorInfoW`) | Click-through; no keyboard focus; no activation | Player Process on exit | `DestroyWindow(hWnd)` returns `TRUE`; VRAM freed |
| **`WIN-02: WebPlayerRender`** | `kinetiwall-player-web` | Player UI Thread | `KinetiWall_WebPlayer_Render` | Child of `WorkerW` (`SetParent`) | `WS_CHILD \| WS_VISIBLE \| WS_CLIPSIBLINGS` / `WS_EX_TRANSPARENT \| WS_EX_NOACTIVATE` | Bound to Display 2 physical rect (`GetMonitorInfoW`) | Click-through (Default); Input capture enabled only on interactive click | Player Process on exit | `DestroyWindow(hWnd)` returns `TRUE`; WebView2 closed |
| **`WIN-03: CoreMessageHost`** | `kinetiwall-core` | Core Main Thread | `KinetiWall_Core_MessageLoop` | Top-level hidden (Message-Only: `HWND_MESSAGE`) | `WS_POPUP` / `0` | Non-visible ($0 \times 0$) | No input; message pump only | Core Daemon on shutdown | `DestroyWindow(hWnd)` returns `TRUE` |
| **`WIN-04: SettingsLibraryUI`** | `kinetiwall-ui` | UI GUI Thread | `KinetiWall_UI_MainWindow` | Top-level standard desktop window | `WS_OVERLAPPEDWINDOW` / `WS_EX_APPWINDOW` | Centered on primary display | Normal user interactive window | UI Process on close (`WM_CLOSE`) | `ExitProcess(0)`; RAM drops to $0.0\text{ MB}$ |
| **`WIN-05: WorkerW_Host`** | `explorer.exe` (External) | Explorer Shell Thread | `WorkerW` | Top-level Windows Shell window | `WS_POPUP \| WS_VISIBLE \| WS_CLIPCHILDREN` | Spans full virtual desktop | Receives shell events; passes clicks to icons | OS Explorer Lifecycle | Never destroyed by KinetiWall |
| **`WIN-06: Progman`** | `explorer.exe` (External) | Explorer Shell Thread | `Progman` | Top-level Windows Shell window | `WS_POPUP \| WS_CLIPCHILDREN` | Full primary display | Shell desktop manager | OS Explorer Lifecycle | Never destroyed by KinetiWall |

---

## 3. Cross-Process Window Embedding (`SetParent`) Semantics

Reparenting a window across process boundaries using Win32 `SetParent(hChild, hParent)` involves complex platform mechanics that must be rigorously managed:

### 3.1 Thread Attachment and Message Queues

* **Win32 Architecture Rule:** Calling `SetParent` across processes synchronizes the input queues of both threads if the child window accepts input.
* **KinetiWall Mitigation:** To prevent KinetiWall from stalling the Windows Explorer message loop (or vice versa), the player window is configured with `WS_EX_TRANSPARENT | WS_EX_NOACTIVATE`. Windows does not route mouse or keyboard input to transparent windows, preventing queue synchronization overhead and eliminating Explorer deadlock risks.

### 3.2 Coordinate Space Transformation

* Prior to `SetParent`, coordinates are in **virtual screen coordinates** ($X, Y$).
* After `SetParent(hPlayerWnd, hWorkerW)`, coordinates become **client coordinates relative to WorkerW**. Because `WorkerW` spans the entire virtual desktop starting at $(0, 0)$ relative to its top-left bound, the target monitor coordinates must be normalized:
  $$\text{ClientX} = \text{MonitorRect.left} - \text{WorkerW.VirtualLeft}$$
  $$\text{ClientY} = \text{MonitorRect.top} - \text{WorkerW.VirtualTop}$$

### 3.3 DPI Awareness Synchronization

* When a child window is embedded in a parent with differing DPI awareness, Windows can produce blurry rendering or clipping.
* `kinetiwall-player-*.exe` processes declare `DPI_AWARENESS_CONTEXT_PER_MONITOR_AWARE_V2` in their application manifests, ensuring exact physical pixel mapping to `WorkerW` regardless of the host Explorer DPI scaling.

---

## 4. Window Validation Protocols

To guarantee that KinetiWall never reparents into or manipulates an invalid, recycled, or unrelated third-party window:

```rust
pub unsafe fn validate_desktop_host(h_target: HWND) -> Result<(), HostValidationError> {
    // Check 1: Handle validity
    if !IsWindow(h_target).as_bool() {
        return Err(HostValidationError::InvalidHandle);
    }

    // Check 2: Verify Window Class is WorkerW
    let mut class_name = [0u16; 64];
    let len = RealGetWindowClassW(h_target, &mut class_name);
    let class_str = String::from_utf16_lossy(&class_name[..len as usize]);
    if class_str != "WorkerW" {
        return Err(HostValidationError::UnexpectedWindowClass(class_str));
    }

    // Check 3: Verify Process Ownership belongs to explorer.exe
    let mut pid: u32 = 0;
    GetWindowThreadProcessId(h_target, Some(&mut pid));
    if !is_explorer_process(pid) {
        return Err(HostValidationError::NonExplorerOwner(pid));
    }

    // Check 4: Verify Terminal Session ID matches active user
    let current_session = get_current_session_id();
    let mut target_session: u32 = 0;
    ProcessIdToSessionId(pid, &mut target_session);
    if current_session != target_session {
        return Err(HostValidationError::CrossSessionForbidden);
    }

    Ok(())
}
```

