# Windows Desktop Integration Compatibility Matrix & Test Matrix
**Document ID:** `KW-DSK-005`  
**Deliverable:** E (Windows Desktop Integration Compatibility Matrix)  
**Parent Document:** [README.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/desktop_integration/README.md)  
**Classification:** Quality Assurance & System Verification Specification  

---

## 1. Operating System Compatibility & Support Matrix

This matrix establishes the verified, supported, and unverified platform configurations for KinetiWall's desktop hosting architecture based on the platform policy established in Step 4:

| Windows OS Edition & Build | Arch | Shell Host Behavior | WorkerW 0x052C Support | Multi-Monitor Status | Integration Confidence | Current Verification Status | Release Blocking? |
| :--- | :---: | :---: | :---: | :---: | :---: | :---: | :---: |
| **Windows 10 21H2 (19044)** | x64 | Standard `Progman` / `WorkerW` | Verified Functional | Full Support | High | `NOT RUN (DESIGN PHASE)` | **Yes (Core Release Target)** |
| **Windows 10 22H2 (19045)** | x64 | Standard `Progman` / `WorkerW` | Verified Functional | Full Support | High | `NOT RUN (DESIGN PHASE)` | **Yes (Core Release Target)** |
| **Windows 11 21H2 (22000)** | x64 | Standard `Progman` / `WorkerW` | Verified Functional | Full Support | High | `NOT RUN (DESIGN PHASE)` | **Yes (Core Release Target)** |
| **Windows 11 22H2 (22621)** | x64 | Standard `Progman` / `WorkerW` | Verified Functional | Full Support | High | `NOT RUN (DESIGN PHASE)` | **Yes (Core Release Target)** |
| **Windows 11 23H2 (22631)** | x64 | Standard `Progman` / `WorkerW` | Verified Functional | Full Support | High | `NOT RUN (DESIGN PHASE)` | **Yes (Core Release Target)** |
| **Windows 11 24H2 (26100)** | x64 | Standard `Progman` / `WorkerW` | Verified Functional | Full Support | High | `NOT RUN (DESIGN PHASE)` | **Yes (Core Release Target)** |
| **Windows 11 ARM64** | ARM64 | Standard `Progman` / `WorkerW` | Expected Functional | Full Support | Medium (Emulation/Native) | `NOT RUN (DESIGN PHASE)` | No (Post-1.0 Target) |
| **Windows Server 2022/2025** | x64 | Desktop Experience Role req. | Conditional | Limited | Low | `NOT RUN (DESIGN PHASE)` | No (Explicit Non-Goal) |
| **Windows 7 / 8.1** | x64 | Legacy Shell | Deprecated | Unsupported | None | **UNSUPPORTED** | No (Explicit Non-Goal) |

---

## 2. Desktop Integration Verification Test Matrix (`DSK-001` through `DSK-015`)

| Test ID | Test Scenario | Tested Subsystems | Pass / Fail Acceptance Criteria | Status |
| :--- | :--- | :--- | :--- | :--- |
| **`DSK-001`** | **Desktop Icon Layer Verification** | `SysListView32` & `WorkerW` | Desktop icons (`Recycle Bin`, files) render visually on top of the live video wallpaper. Zero icon occlusion or tearing. | `NOT RUN (DESIGN PHASE)` |
| **`DSK-002`** | **Desktop Selection & Marquee Drag** | Shell Desktop Input | Clicking and dragging a rubber-band marquee rectangle on desktop selects icons smoothly without video stutter. | `NOT RUN (DESIGN PHASE)` |
| **`DSK-003`** | **Desktop Shell Context Menu** | Explorer Shell Context Menu | Right-clicking on wallpaper surfaces instantly displays the standard Windows desktop context menu (`View`, `Sort by`, `New`). | `NOT RUN (DESIGN PHASE)` |
| **`DSK-004`** | **Taskbar & Start Menu Z-Order** | `Shell_TrayWnd` & Shell | Taskbar, Start menu, Action Center, and system notifications remain strictly on top of wallpaper across all displays. | `NOT RUN (DESIGN PHASE)` |
| **`DSK-005`** | **Zero Focus Stealing** | Win32 Window Activation | Clicking anywhere on live wallpaper never steals keyboard focus from active text editor or foreground application. | `NOT RUN (DESIGN PHASE)` |
| **`DSK-006`** | **Multi-Monitor Mixed DPI Placement** | Virtual Screen / Mixed DPI | Primary display at 4K 150% DPI + secondary at 1080p 100% DPI render pixel-perfect wallpapers without blur or boundary overlap. | `NOT RUN (DESIGN PHASE)` |
| **`DSK-007`** | **Negative Virtual Coordinate Handling**| Monitor Positioning | Secondary monitor positioned at $[-1920, 0]$ (left of primary) maps child HWND coordinates correctly without clipping. | `NOT RUN (DESIGN PHASE)` |
| **`DSK-008`** | **Explorer Crash Recovery** | Shell Lifecycle (`TaskbarCreated`)| Killing `explorer.exe` via Task Manager triggers automatic recovery: Core re-anchors players within $\le 500\text{ ms}$ upon shell restart. | `NOT RUN (DESIGN PHASE)` |
| **`DSK-009`** | **Display Resolution Hot-Change** | `WM_DISPLAYCHANGE` | Changing resolution from 1080p to 1440p dynamically resizes DXGI swapchain buffers via `ResizeBuffers` without restarting player. | `NOT RUN (DESIGN PHASE)` |
| **`DSK-010`** | **Monitor Disconnect / Hotplug** | Multi-Monitor Topology | Unplugging HDMI cable cleanly terminates target player; plugging cable back in restores wallpaper within $1000\text{ ms}$. | `NOT RUN (DESIGN PHASE)` |
| **`DSK-011`** | **Fullscreen Gaming Occlusion** | Hardware Occlusion / Pause | Launching a fullscreen DX12/Vulkan game pauses wallpaper swapchain rendering, dropping background GPU usage to $0.0\%$. | `NOT RUN (DESIGN PHASE)` |
| **`DSK-012`** | **Workstation Lock / Sleep Quiescence** | Power & Session Monitor | Locking workstation (`Win+L`) or monitor sleep pauses video decoders; CPU drops to $< 0.1\%$; resumes smoothly on unlock. | `NOT RUN (DESIGN PHASE)` |
| **`DSK-013`** | **Clean Shell Desktop Teardown** | Shutdown / Player Stop | Stopping wallpaper restores native Windows static wallpaper bitmap instantly without screen flash or black rectangles. | `NOT RUN (DESIGN PHASE)` |
| **`DSK-014`** | **Host Discovery Deadlock Immunity** | `SendMessageTimeoutW` | Injecting an artificial 5-second hang in Explorer does not hang Core thread; timeout triggers at exactly $1000\text{ ms}$. | `NOT RUN (DESIGN PHASE)` |
| **`DSK-015`** | **Ambiguous Hierarchy Fallback** | Shell Validation Fallback | Corrupting shell window classes causes Core to gracefully drop to `INTEGRATION_DEGRADED` without crashing or looping. | `NOT RUN (DESIGN PHASE)` |

---

## 3. Automated Test Harness Specification

Automated testing of desktop integration in CI environments requires running inside a dedicated Windows Virtual Machine with the **Windows Indirect Display Driver (IDD)** to simulate multi-monitor topologies:

```
+-------------------------------------------------------------------------+
|                  CI Test Pipeline: Windows Server / Win11 VM             |
+-------------------------------------------------------------------------+
|  1. Virtual Display Setup:                                              |
|     - Spawns 2 virtual displays: [3840x2160 @ 150%] + [1920x1080 @ 100%]|
|  2. Harness Executor (`cargo test --test desktop_integration`):         |
|     - Launches kinetiwall-core and players                              |
|  3. Win32 Window Hierarchy Auditor:                                     |
|     - Traverses EnumChildWindows on Progman and WorkerW                 |
|     - Asserts SysListView32 z-order > KinetiWall_VideoPlayer_Render     |
|  4. Synthetic Input Injector:                                           |
|     - Uses SendInput to perform mouse clicks on desktop icons           |
|     - Verifies icon selection state changes in SysListView32            |
|  5. Explorer Chaos Injector:                                            |
|     - Issues taskkill /F /IM explorer.exe                               |
|     - Evaluates recovery time to INTEGRATED state                       |
+-------------------------------------------------------------------------+
```

