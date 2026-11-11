# KinetiWall — Compatibility Test & Verification Matrix

**Document Identifier:** KW-POL-005  
**Document Type:** Formal Engineering Compatibility Test Protocol (Deliverable E)  
**Current Phase:** Phase 1 — Project Research, Requirements & Feasibility  
**Step:** Step 4 of 90 — Windows Compatibility & Platform Support Policy  
**Status:** Approved Test Protocol Baseline (Planned Verification)  
**Date of Research:** 2026-11-08  

---

## 1. Testing Protocol Governance & Standards

To establish an evidence-backed platform support claim, this test matrix formalizes **twenty comprehensive verification scenarios** (`CT-001` to `CT-020`) spanning the entire application lifecycle, operating system matrix, hardware transitions, and failure modes.

### Verification Status Rule:
In strict compliance with Step 4 engineering policy:
- Because runtime feature implementation has not commenced, all test execution statuses are explicitly designated as **`NOT RUN (PLANNED)`**.
- No test is recorded as passed without actual empirical execution on verified physical or virtual test benches during Phase 2.
- A platform or operating system build can only achieve **Tier A Primary Supported** status when all twenty test scenarios pass with zero Blocker or Critical defects.

### Defect Severity Definitions:
- **BLOCKER:** Core functionality fails completely (crash on launch, black desktop, shell freeze, broken icon clicks). Blocks official support.
- **CRITICAL:** High-priority feature failure (video stutter, pause detection failure, memory leak >50 MB). Blocks release.
- **MAJOR:** Feature degraded but functional workaround exists (e.g. software decode fallback active, non-critical tray glitch).
- **MINOR:** Cosmetic or logging defect (e.g. minor UI font scaling imperfection).

---

## 2. Master Compatibility Test Matrix (CT-001 to CT-020)

| Test ID | Test Scenario | Target Environment & Prerequisites | Required Execution Steps | Expected Observable Outcome | Evidence to Collect | Failure Severity | Blocks Official Support? | Execution Status |
| :--- | :--- | :--- | :--- | :--- | :--- | :---: | :---: | :---: |
| **CT-001** | **Clean Installation** | Win 10 (22H2) & Win 11 (23H2/24H2), Non-admin user account | 1. Run Inno Setup installer.<br>2. Select install directory.<br>3. Complete wizard. | Installs without UAC prompt (`asInvoker`); creates desktop shortcut and tray icon; zero left-over temp files. | Install logs, exit code, registry entries in `HKCU`. | **BLOCKER** | **YES** | `NOT RUN (PLANNED)` |
| **CT-002** | **Upgrade Installation** | Pre-existing KinetiWall v1.0 running with active wallpaper | 1. Execute newer version installer over running instance. | Gracefully signals active daemon via IPC to exit; replaces binaries; restores active wallpaper upon relaunch. | File replacement log, version query via CLI. | **CRITICAL** | **YES** | `NOT RUN (PLANNED)` |
| **CT-003** | **Startup & Clean Shutdown** | Win 10 & Win 11 x86-64 | 1. Launch `kinetiwall-core.exe`.<br>2. Verify tray presence.<br>3. Select "Exit" from menu. | Daemon initializes in <500ms; on exit, cleanly destroys all windows and restores original desktop wallpaper. | Process lifetime monitor, desktop wallpaper visual inspection. | **BLOCKER** | **YES** | `NOT RUN (PLANNED)` |
| **CT-004** | **Sub-Desktop Window Attachment** | Windows 11 24H2 (Build 26100) with desktop icons enabled | 1. Launch live video wallpaper.<br>2. Inspect window hierarchy.<br>3. Click desktop icons. | Wallpaper renders behind desktop icons; drag-select marquee works; right-click context menu opens normally. | Win32 Spy++ window tree, synthetic mouse click event logs. | **BLOCKER** | **YES** | `NOT RUN (PLANNED)` |
| **CT-005** | **Hidden Desktop Icons Attachment** | Windows 11 24H2 with "Show desktop icons" disabled | 1. Uncheck "Show desktop icons" in Windows.<br>2. Launch live wallpaper. | Dual-mode algorithm detects hidden icons; parents beneath `Progman` without infinite flashing or grey screen. | Frame capture video, DWM surface event logs. | **BLOCKER** | **YES** | `NOT RUN (PLANNED)` |
| **CT-006** | **Windows Explorer Restart Recovery** | Win 10 / Win 11 during active video playback | 1. Execute `Stop-Process -Name explorer -Force` in PowerShell.<br>2. Observe recovery. | Intercepts `TaskbarCreated`; waits 500ms; re-parents wallpaper window without crashing or leaking GPU context. | PowerShell process log, desktop restore timestamp (<2.0s). | **CRITICAL** | **YES** | `NOT RUN (PLANNED)` |
| **CT-007** | **Hardware Video Decode & Seamless Loop** | 1080p60 H.264 & 4K WebM VP9 video loops | 1. Play video wallpaper continuously for 50 loop transitions. | Direct3D 11 hardware decoding active (`d3d11va`); loop boundary jitter <16.6ms; zero black frame flashes. | Frame-timing profiler CSV, GPU engine utilization counter. | **CRITICAL** | **YES** | `NOT RUN (PLANNED)` |
| **CT-008** | **Software Decoding Fallback** | System with disabled or simulated broken GPU hardware decode | 1. Force software decode flag in config.<br>2. Load video wallpaper. | Fallback to CPU decode (`dav1d`/`libavcodec`) succeeds; logs warning; video plays smoothly without crashing. | Diagnostic log file containing software fallback warning. | **MAJOR** | **NO** (Fallback functional) | `NOT RUN (PLANNED)` |
| **CT-009** | **WebView2 Runtime Initialization** | Windows 11 23H2 with Evergreen WebView2 | 1. Apply interactive HTML5/WebGL wallpaper. | `CoreWebView2Controller` embeds behind desktop icons; WebGL2 particles render at steady 60 FPS. | WebGL frame counter, JS console log (zero errors). | **BLOCKER** | **YES** | `NOT RUN (PLANNED)` |
| **CT-010** | **WebView2 Missing Runtime Handling** | Win 10 1903 clean VM without WebView2 preinstalled | 1. Launch KinetiWall on system lacking WebView2.<br>2. Attempt web wallpaper. | Web wallpapers disabled with clear UI prompt; video playback remains fully operational; zero app crash. | UI error dialog screenshot, registry query log. | **CRITICAL** | **YES** | `NOT RUN (PLANNED)` |
| **CT-011** | **Fullscreen Application Pause** | 3D Game / DirectX 11 benchmark launched in fullscreen | 1. Launch 3D game in exclusive/borderless fullscreen. | Fullscreen detected via `SetWinEventHook` within 100ms; video rendering pauses; CPU drops to 0.0%. | Event hook timestamp log, CPU thread trace. | **BLOCKER** | **YES** | `NOT RUN (PLANNED)` |
| **CT-012** | **Multi-Monitor Independent Assignment** | Dual-Monitor workstation (1080p primary + 4K secondary) | 1. Assign Video A to Monitor 1.<br>2. Assign Web B to Monitor 2. | Both monitors render distinct content with correct aspect ratios; audio streams only from primary. | Screenshot of dual-display workspace, display HWND audit. | **CRITICAL** | **YES** | `NOT RUN (PLANNED)` |
| **CT-013** | **Multi-Monitor Spanning Layout** | Dual-Monitor ultra-wide topology (3840×1080 combined) | 1. Enable "Span across displays" mode with panoramic video. | Single unified video canvas spans across virtual screen bounds; correct coordinate offsetting. | Virtual screen metric logs, alignment inspection. | **MAJOR** | **YES** | `NOT RUN (PLANNED)` |
| **CT-014** | **Mixed-DPI & Resolution Changes** | 200% DPI laptop display connected to 100% DPI desktop monitor | 1. Move window across displays.<br>2. Change display resolution in Windows. | Intercepts `WM_DPICHANGED` and `WM_DISPLAYCHANGE`; viewports scale cleanly without blurriness or coordinate drift. | High-DPI physical coordinate capture, DWM blur audit. | **CRITICAL** | **YES** | `NOT RUN (PLANNED)` |
| **CT-015** | **Display Hotplug & Removal** | Laptop connected to external HDMI display | 1. Unplug external display during playback.<br>2. Reconnect 5 seconds later. | Detects disconnection; gracefully cleans up secondary player window; recreates player on reconnect within 1.0s. | Windows device change event log, memory leak audit. | **CRITICAL** | **YES** | `NOT RUN (PLANNED)` |
| **CT-016** | **Sleep / Modern Standby Wake Recovery** | Win 10 / Win 11 S3 sleep or S0ix modern standby | 1. Put PC to sleep during video playback.<br>2. Wake PC after 2 minutes. | Intercepts `WM_POWERBROADCAST`; catches `DXGI_ERROR_DEVICE_RESET`; rebuilds swapchain within 2.0s of wake. | Power broadcast event logs, GPU swapchain re-init trace. | **BLOCKER** | **YES** | `NOT RUN (PLANNED)` |
| **CT-017** | **Battery Mode Power Throttling** | Battery-powered laptop on AC power | 1. Disconnect laptop AC charger.<br>2. Reconnect AC charger. | Detects battery mode within 1.0s; pauses live wallpaper or throttles to 30 FPS; resumes on AC reconnect. | Battery status query log, framerate profiler trace. | **CRITICAL** | **YES** | `NOT RUN (PLANNED)` |
| **CT-018** | **WASAPI Audio Endpoint Switching** | Audio-reactive wallpaper active while playing music | 1. Unplug standard speakers.<br>2. Plug in Bluetooth headphones. | Intercepts `OnDefaultDeviceChanged`; re-initializes loopback stream within 500ms; zero audio distortion or crash. | CoreAudio event log, FFT spectrum magnitude stream. | **MAJOR** | **YES** | `NOT RUN (PLANNED)` |
| **CT-019** | **Orphan Process Cleanup via Job Object** | Core daemon supervising 2 active player processes | 1. Force kill `kinetiwall-core.exe` via Task Manager (`kill -f`). | Windows kernel Job Object automatically terminates all child player processes within 100ms; zero zombies. | PowerShell process tree query verifying zero orphan PIDs. | **CRITICAL** | **YES** | `NOT RUN (PLANNED)` |
| **CT-020** | **Complete Offline Installation & Operation**| Air-gapped Windows 10/11 virtual machine (Network Disabled)| 1. Disable all NICs.<br>2. Install KinetiWall.<br>3. Import & play local wallpaper. | Installs, launches, indexes library, and plays video/web wallpapers with 100% functionality; zero network calls. | Wireshark packet capture (0 packets), local database audit. | **BLOCKER** | **YES** | `NOT RUN (PLANNED)` |

---

## 3. Required Physical & Virtual Test Bench Matrix

To certify KinetiWall for Tier A and Tier B release distribution, test execution must cover these five reference test environments:

1. **Environment 1 (Modern Windows 11 Desktop):** Windows 11 Version 24H2 (Build 26100) / x86-64 / Intel Core i7 / NVIDIA GeForce RTX 3070 / Dual 4K Displays (Mixed 150%/100% DPI).
2. **Environment 2 (Standard Windows 11 Laptop):** Windows 11 Version 23H2 (Build 22631) / x86-64 / AMD Ryzen 7 7840U / Radeon 780M iGPU / 16 GB RAM / Battery Power Testing.
3. **Environment 3 (Windows 10 Enterprise Workstation):** Windows 10 Version 22H2 (Build 19045) / x86-64 / Intel Core i5 / Intel UHD Graphics 630 / Single 1080p Display.
4. **Environment 4 (Windows on ARM Copilot+ PC):** Windows 11 Version 24H2 / ARM64 (Qualcomm Snapdragon X Elite) / Prism Emulation Evaluation.
5. **Environment 5 (Air-Gapped Offline VM):** Windows 10 Version 21H2 (Build 19044) / Virtualized Hyper-V / Zero Network Connectivity.

