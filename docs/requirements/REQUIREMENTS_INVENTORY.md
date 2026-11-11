# KinetiWall — Comprehensive Requirements Inventory

**Document Identifier:** KW-DOC-002  
**Document Type:** Formal Requirements Specification  
**Current Phase:** Phase 1 — Project Research, Requirements & Feasibility  
**Step:** Step 1 of 90 — Project Requirements Inventory & Traceability  
**Status:** Approved Working Baseline  
**Date:** 2026-11-08  

---

## 1. Classification & Priority Taxonomy

Every requirement documented in this inventory is assigned an explicit **Classification** and **Priority** in compliance with Step 1 engineering standards.

### 1.1 Classification Definitions
- **USER-REQUIRED:** Explicitly demanded or specified by the user in project requirements or approved briefs. Mandatory for project delivery.
- **TECHNICAL-DECISION:** An architectural or technological decision formally approved to satisfy user requirements.
- **RESEARCH-RECOMMENDATION:** Proposed by technical research (e.g., Gemini benchmarks, competitor analysis) as an optimal pattern; requires empirical validation before final approval.
- **OPTIONAL-ENHANCEMENT:** A desirable capability or feature extension awaiting formal scheduling.
- **FUTURE-RELEASE:** A capability intentionally deferred beyond the v1.0 release baseline.
- **UNRESOLVED:** A requirement containing unresolved architectural questions, ambiguous criteria, or dependencies on unverified platform behaviors.
- **NON-GOAL:** Explicitly excluded functionality or behavior designed to maintain project scope and integrity.

### 1.2 Priority Levels
- **P0:** Essential to the core product mission. Failure to deliver prevents viable operation.
- **P1:** High priority for initial production release. Delivers standard expected capabilities.
- **P2:** Approved secondary capability. Enhances utility without blocking baseline usage.
- **P3:** Lower-priority enhancement or stretch goal for subsequent milestones.
- **UNASSIGNED:** Priority pending user clarification or technical feasibility resolution.

---

## 2. Functional Requirements Inventory (KW-F)

### 2.1 Desktop Integration (Win32 Shell Layer)

#### KW-F-001: Sub-Desktop Window Attachment (WorkerW / Progman)
- **Title:** Native Desktop Window Attachment behind Desktop Icons
- **Classification:** USER-REQUIRED
- **Priority:** P0
- **Status:** PLANNED
- **Source:** User Project Brief, Section 6.1
- **Statement:** The wallpaper player window must render directly behind Windows desktop icons (`SysListView32`) while permitting normal desktop interactions, including icon single/double clicking, multi-selection bounding boxes, desktop right-click context menus, and shortcut launching.
- **Rationale:** Core live wallpaper functionality requires rendering behind shell icons without capturing or interfering with standard desktop mouse and keyboard interactions.
- **Acceptance Criteria:**
  1. The rendering window handle (`HWND`) is parented or positioned beneath the desktop icon listview (`SHELLDLL_DefView`).
  2. Clicking or dragging over desktop icons selects the icons and shows the standard Windows marquee selection rectangle.
  3. Right-clicking an empty desktop area invokes the standard Windows Explorer context menu.
- **Verification Method:** Manual and automated Win32 GUI testing checking window Z-order (`GetWindow`, `GW_HWNDPREV`/`GW_HWNDNEXT`), hit testing, and synthetic mouse click/drag events over desktop icons.
- **Dependencies:** Windows Win32 API (`user32.dll`), `Progman` message handling.
- **Risks & Constraints:** Relies on the undocumented Windows message `0x052C` sent to `Progman` to spawn a `WorkerW` window. Microsoft does not guarantee the longevity or stability of this message across Windows updates.

#### KW-F-002: Windows Explorer Restart Recovery
- **Title:** Dynamic Re-attachment on Explorer Crash or Restart
- **Classification:** USER-REQUIRED
- **Priority:** P0
- **Status:** PLANNED
- **Source:** User Project Brief, Section 6.1
- **Statement:** When `explorer.exe` crashes, terminates, or restarts, KinetiWall must detect the event, invalidate obsolete desktop window handles, re-negotiate desktop attachment, and restore active wallpaper playback without requiring a manual restart of KinetiWall.
- **Rationale:** In production desktop use, shell restarts occur during graphic driver updates, Windows Explorer updates, or shell crashes. Orphaned rendering windows degrade desktop usability.
- **Acceptance Criteria:**
  1. The core controller registers the `TaskbarCreated` Win32 registered window message.
  2. Upon receiving `TaskbarCreated` or detecting `Progman` handle invalidation, active player window handles are detached or gracefully recreated.
  3. Desktop integration is re-established within 2.0 seconds of `explorer.exe` relaunch without leaking window handles or GPU surfaces.
- **Verification Method:** Automated test script terminating `explorer.exe` via `Stop-Process -Name explorer -Force` during active video playback, validating re-attachment and render resumption.
- **Dependencies:** KW-F-001, Win32 Message Loop (`RegisterWindowMessageW`).
- **Risks & Constraints:** During Explorer initialization, rapid window hierarchy changes can cause race conditions during `FindWindowExW` calls. Bounded retry backoff is required.

#### KW-F-003: Graceful Attachment Failure & Fallback
- **Title:** Desktop Attachment Fallback Mode on Shell Rejection
- **Classification:** RESEARCH-RECOMMENDATION
- **Priority:** P1
- **Status:** PLANNED
- **Source:** Research Document, Section 6.1
- **Statement:** If the engine fails to spawn or attach to a `WorkerW` window (e.g., due to third-party shell replacements or restrictive security policies), KinetiWall must log the failure, notify the user with an actionable message, and safely fallback to either standard borderless window mode or restore the native Windows static wallpaper without crashing.
- **Rationale:** Prevents application panic and corrupted desktop states when interacting with non-standard Windows desktop environments.
- **Acceptance Criteria:**
  1. Attachment routine incorporates a 3-attempt bounded retry with 250ms backoff.
  2. If attachment fails after retries, the original desktop wallpaper is restored via `SystemParametersInfoW(SPI_SETDESKWALLPAPER)`.
  3. A diagnostic notification is posted to the system tray log.
- **Verification Method:** Unit test injecting simulated attachment failure returns; verify clean error logging and absence of unhandled exceptions.
- **Dependencies:** KW-F-001.
- **Risks & Constraints:** None.

#### KW-F-004: Clean Application Termination & Desktop Restoration
- **Title:** Shell State Restoration on Application Exit
- **Classification:** USER-REQUIRED
- **Priority:** P0
- **Status:** PLANNED
- **Source:** User Project Brief, Section 6.1
- **Statement:** Upon application shutdown, graceful exit, or crash, KinetiWall must close all player windows, release device contexts, and signal the Windows shell to redraw the standard desktop background without leaving black screens, frozen artifacts, or orphaned windows.
- **Rationale:** Users must never experience desktop corruption when stopping or uninstalling live wallpapers.
- **Acceptance Criteria:**
  1. Interception of `WM_CLOSE`, `WM_QUERYENDSESSION`, `WM_ENDSESSION`, and `Ctrl+C` / `SIGTERM` signals.
  2. All child/worker windows are cleanly destroyed (`DestroyWindow`).
  3. `RedrawWindow` or `SystemParametersInfoW` is dispatched to refresh the desktop surface.
- **Verification Method:** Shutdown test measuring process termination and visually inspecting desktop surface to verify standard wallpaper display.
- **Dependencies:** KW-F-001, KW-F-006.
- **Risks & Constraints:** Abrupt `SIGKILL` / `TerminateProcess` cannot run in-process cleanup; requires core daemon watchdog or supervisor to clean up resources if a player terminates abnormally.

---

### 2.2 Multi-Process Architecture & Crash Isolation

#### KW-F-005: Core Controller Daemon Process
- **Title:** Dedicated Lightweight Native Core Controller
- **Classification:** USER-REQUIRED
- **Priority:** P0
- **Status:** PLANNED
- **Source:** User Project Brief, Section 6.2
- **Statement:** KinetiWall must execute as a lightweight native core controller process (`kinetiwall-core.exe` or equivalent) responsible for configuration management, display monitoring, player process lifecycle supervision, system tray presence, and IPC routing.
- **Rationale:** Decoupling management and orchestration from rendering guarantees stability, rapid startup, and minimal idle footprint.
- **Acceptance Criteria:**
  1. The core controller runs as a standalone native Rust executable.
  2. It maintains a system tray icon with an actionable context menu.
  3. It continuously monitors child player processes and handles OS power/display events.
- **Verification Method:** Process inspection via `Get-Process`; verify executable footprint and operational responsiveness.
- **Dependencies:** Win32 API, Rust Standard Library.
- **Risks & Constraints:** Must ensure thread safety across event listeners and IPC server threads.

#### KW-F-006: Isolated Wallpaper Player Processes
- **Title:** Out-of-Process Wallpaper Rendering Engines
- **Classification:** USER-REQUIRED
- **Priority:** P0
- **Status:** PLANNED
- **Source:** User Project Brief, Section 6.2
- **Statement:** Video playback, web rendering, and shader execution must occur in isolated child processes (`kinetiwall-player.exe` or engine-specific binaries) running out-of-process from the core controller daemon.
- **Rationale:** Complex media decoders (libmpv/FFmpeg) and browser runtimes (WebView2) are susceptible to memory leaks, third-party codec crashes, and GPU driver faults. Isolation prevents wallpaper faults from crashing the core manager or Windows desktop.
- **Acceptance Criteria:**
  1. Video playback and web rendering run in separate OS process spaces from the core controller.
  2. Terminating a player process via Task Manager does not terminate `kinetiwall-core`.
  3. The core controller detects player termination within 500ms and executes configured restart or notification policies.
- **Verification Method:** Process kill test (`Stop-Process -Id <player_pid> -Force`); verify core controller remains active and logs crash event.
- **Dependencies:** KW-F-005, Windows Process APIs (`CreateProcessW`, `JobObject`).
- **Risks & Constraints:** Inter-process window parenting requires precise Win32 handle passing and synchronization.

#### KW-F-007: On-Demand Gallery & Settings User Interface
- **Title:** Ephemeral On-Demand UI Process
- **Classification:** USER-REQUIRED
- **Priority:** P0
- **Status:** PLANNED
- **Source:** User Project Brief, Section 6.2, 6.11
- **Statement:** The wallpaper gallery, browser, and settings user interface must launch as an on-demand process that completely terminates and unloads all memory when closed by the user. It must **never** linger as a hidden, resident background process.
- **Rationale:** UIs built on modern frameworks (e.g., Tauri/WebView2) consume 50–150 MB of RAM. Keeping the UI loaded while idle wastes system memory, directly violating KinetiWall's lightweight philosophy.
- **Acceptance Criteria:**
  1. Closing the gallery window (via titlebar 'X', Alt+F4, or menu exit) invokes a clean process exit of the UI binary.
  2. `Get-Process` confirms the UI process is fully terminated within 1.0 second of closing.
  3. Wallpaper playback continues uninterrupted in the player processes supervised by `kinetiwall-core`.
- **Verification Method:** Automated test opening and closing the gallery UI while asserting running processes and memory before, during, and after UI presence.
- **Dependencies:** KW-F-005, KW-NG-001.
- **Risks & Constraints:** Requires state persistence (window position, filters) saved cleanly to disk prior to UI exit.

#### KW-F-008: Inter-Process Communication (IPC) Engine
- **Title:** Secure High-Performance Local Named Pipe IPC
- **Classification:** TECHNICAL-DECISION
- **Priority:** P0
- **Status:** PLANNED
- **Source:** Approved Project Decisions & Section 6.2
- **Statement:** The core controller, UI process, and player instances must communicate using Windows Named Pipes (`\\.\pipe\kinetiwall-ipc-*`) with strict schema-validated, versioned JSON-RPC or binary-serialized message payloads.
- **Rationale:** Named Pipes provide high-throughput, low-latency, kernel-mediated local IPC with native Windows security descriptor support.
- **Acceptance Criteria:**
  1. Named Pipe listener initializes with security descriptors restricted to the current user's security identifier (SID).
  2. IPC messages validate against a versioned message schema.
  3. Malformed, oversized, or unauthorized connection attempts are immediately dropped and logged.
  4. Core round-trip latency for state commands (Play, Pause, Volume) is less than 15 ms.
- **Verification Method:** IPC integration test exercising pipe connection, schema validation, message throughput, and rejection of invalid payloads.
- **Dependencies:** Windows Named Pipe API, `serde` / serialization crate.
- **Risks & Constraints:** Pipe buffer sizing must accommodate audio FFT arrays without blocking command message queues.

#### KW-F-009: Player Crash Supervision & Bounded Exponential Backoff
- **Title:** Bounded Restart Supervision for Failed Players
- **Classification:** TECHNICAL-DECISION
- **Priority:** P1
- **Status:** PLANNED
- **Source:** Research Document, Section 6.2 & 7.2
- **Statement:** If a player process crashes repeatedly, the core controller must enforce a bounded restart policy (e.g., maximum 3 restarts within 60 seconds) with exponential backoff before marking the wallpaper as faulted and pausing automatic retries.
- **Rationale:** Prevents infinite crash/restart loops that peg CPU, flood event logs, and destabilize the host system.
- **Acceptance Criteria:**
  1. Player restarts are tracked per display instance.
  2. If a player crashes 3 times within 60 seconds, further automatic restarts are suspended.
  3. A diagnostic notification is dispatched to the user offering to inspect logs or reset wallpaper.
- **Verification Method:** Automated test invoking a deliberately crashing mock player; assert restart count halts at 3 and trigger notification.
- **Dependencies:** KW-F-005, KW-F-006.
- **Risks & Constraints:** Must ensure restart counters reset upon continuous successful playback exceeding 5 minutes.

---

### 2.3 Wallpaper Playback & Media Engine

#### KW-F-010: Native Video Container Support (MP4, WebM, AVI)
- **Title:** Core Video Format Decoding
- **Classification:** USER-REQUIRED
- **Priority:** P0
- **Status:** PLANNED
- **Source:** User Project Brief, Section 6.3
- **Statement:** The video player engine must decode and display video files in MP4 (`.mp4`), WebM (`.webm`), and AVI (`.avi`) containers with common codecs (H.264/AVC, H.265/HEVC, VP8, VP9, AV1 where hardware supported).
- **Rationale:** MP4 and WebM represent >90% of user live wallpapers and online video clips; AVI satisfies legacy asset playback.
- **Acceptance Criteria:**
  1. Successful playback of verified test suite files: `test_1080p.mp4` (H.264), `test_1080p.webm` (VP9), and `test_720p.avi` (MPEG-4).
  2. Render output scales correctly to display resolution without aspect ratio distortion.
- **Verification Method:** Automated media test suite loading reference video files and verifying successful frame presentation via decoder timestamps.
- **Dependencies:** Video player engine (libmpv or Windows Media Foundation).
- **Risks & Constraints:** Codec availability on minimal Windows N/KN editions without Media Feature Pack.

#### KW-F-011: Extended Container Format Evaluation (MKV)
- **Title:** Extended Matroska Video Support
- **Classification:** RESEARCH-RECOMMENDATION
- **Priority:** P2
- **Status:** PLANNED
- **Source:** Research Document, Section 6.3
- **Statement:** The video engine should support playback of Matroska (`.mkv`) container files, subject to licensing and media decoder capability.
- **Rationale:** MKV is widely used in enthusiast desktop media collections and supports advanced subtitles and audio tracks.
- **Acceptance Criteria:**
  1. MKV files containing H.264/HEVC/VP9 streams load and play with standard looping and transport controls.
  2. Unsupported internal tracks (e.g., complex styled ASS subtitles) do not crash the engine.
- **Verification Method:** Media verification test loading reference `.mkv` files and asserting successful playback.
- **Dependencies:** KW-F-010.
- **Risks & Constraints:** If using Windows Media Foundation, MKV requires third-party splitters or Windows 10 native MKV source reader which has known codec limitations compared to libmpv/FFmpeg.

#### KW-F-012: Hardware-Accelerated Video Decoding
- **Title:** Direct3D / DXVA Hardware Acceleration
- **Classification:** USER-REQUIRED
- **Priority:** P0
- **Status:** PLANNED
- **Source:** User Project Brief, Section 6.3 & Section 7.1
- **Statement:** Video decoding must utilize hardware acceleration (D3D11VA, DXVA2, or vendor-specific GPU decoders) by default to offload video decompression from the host CPU to the GPU.
- **Rationale:** Software decoding of 1080p60 or 4K video consumes 15–40% CPU, generating heat and draining laptop batteries. Hardware decoding keeps CPU overhead under 2–3%.
- **Acceptance Criteria:**
  1. Video playback initializes hardware decoding context (e.g., `d3d11va` in libmpv or D3D11 video decoder in WMF).
  2. CPU utilization during 1080p60 H.264 playback remains below 3.0% on reference test hardware.
  3. Video rendering uses Direct3D swapchains with zero CPU-side memory copies where supported.
- **Verification Method:** Performance test benchmarking CPU and GPU utilization during 60-second 1080p60 playback comparing hardware vs software mode.
- **Dependencies:** Direct3D 11 runtime, graphics hardware supporting relevant codec profiles.
- **Risks & Constraints:** Older integrated GPUs or virtual machine display adapters may fail hardware context initialization, requiring seamless software fallback.

#### KW-F-013: Seamless Looping Playback
- **Title:** Gapless Seamless Video Looping
- **Classification:** USER-REQUIRED
- **Priority:** P0
- **Status:** PLANNED
- **Source:** User Project Brief, Section 6.3
- **Statement:** The video player must provide seamless looping of live wallpaper clips, eliminating black frame flashes, visible stutter, audio pops, or significant frame drops across the loop boundary.
- **Rationale:** Wallpaper videos are short repeating loops (typically 5–30 seconds). A visible black glitch or stutter at every loop ruins visual immersion.
- **Acceptance Criteria:**
  1. Transition from the last frame to the first frame occurs with less than 1 frame display jitter (<16.6ms at 60Hz).
  2. No black or cleared frames are presented to the swapchain across the loop transition.
- **Verification Method:** Frame-timing instrumentation measuring inter-frame intervals across 50 consecutive loop transitions with zero frames exceeding 33.3ms threshold.
- **Dependencies:** KW-F-010, KW-F-012.
- **Risks & Constraints:** Requires decoder pre-buffering or dual-pipeline ping-pong buffering if the underlying decoder has seek/flush latency at end-of-file.

#### KW-F-014: Video Playback Controls (Transport & Audio)
- **Title:** Playback Transport, Seeking, and Volume Control
- **Classification:** USER-REQUIRED
- **Priority:** P1
- **Status:** PLANNED
- **Source:** User Project Brief, Section 6.3
- **Statement:** The player must provide responsive playback controls via IPC: Play, Pause, Stop, Seek to position, Volume level adjustment (0–100%), and Mute toggle.
- **Rationale:** Users must be able to pause wallpaper when working, silence audio tracks, or test specific segments.
- **Acceptance Criteria:**
  1. Dispatching a `Pause` IPC command halts playback within 50 ms and holds the current frame.
  2. Dispatching `SetVolume(x)` adjusts audio output linearly/logarithmically without audio distortion or popping.
  3. Default configuration defaults video wallpaper audio to muted (0% volume) unless explicitly enabled by user.
- **Verification Method:** Automated IPC integration test issuing transport commands and verifying decoder state and audio mixer levels.
- **Dependencies:** KW-F-008, KW-F-010.
- **Risks & Constraints:** None.

#### KW-F-015: Graceful Decoder Fallback on Hardware Failure
- **Title:** Automatic Software Decoding Fallback
- **Classification:** USER-REQUIRED
- **Priority:** P1
- **Status:** PLANNED
- **Source:** User Project Brief, Section 6.3 & 7.2
- **Statement:** If hardware-accelerated decoding fails during initialization or runtime (e.g., driver crash or unsupported profile), the player must automatically fall back to software decoding, log a warning, and continue playback without user-visible disruption.
- **Rationale:** Ensures media resilience across varied and legacy graphics hardware.
- **Acceptance Criteria:**
  1. Detection of hardware context initialization failure triggers immediate fallback to CPU decoding.
  2. Playback initiates successfully without crashing the player process.
  3. Warning message recorded in diagnostic log.
- **Verification Method:** Simulated failure test disabling hardware acceleration flags in configuration; verify video still renders via software pipeline.
- **Dependencies:** KW-F-010, KW-F-012.
- **Risks & Constraints:** Software decoding may exhibit higher CPU load; UI should indicate software fallback state in diagnostics.

---

### 2.4 Interactive Web Wallpapers

#### KW-F-016: HTML5, CSS & JavaScript Web Wallpaper Runtime
- **Title:** Interactive Web Runtime Integration
- **Classification:** USER-REQUIRED
- **Priority:** P0
- **Status:** PLANNED
- **Source:** User Project Brief, Section 6.4
- **Statement:** KinetiWall must provide an isolated web wallpaper player capable of rendering modern HTML5, CSS animations, JavaScript, Canvas 2D, WebGL, and WebGL2 interactive wallpapers.
- **Rationale:** Enables rich interactive experiences, including procedural animations, generative art, and web-based clocks/widgets.
- **Acceptance Criteria:**
  1. Successful execution and rendering of modern web standards including WebGL2 shaders and CSS flex/grid animations.
  2. Rendering achieves up to 60 FPS (or display refresh rate) on capable hardware.
- **Verification Method:** Automated test loading a WebGL2 particle simulation test page; assert steady 60 FPS presentation and zero JavaScript console errors.
- **Dependencies:** Microsoft Edge WebView2 runtime or embedded web engine.
- **Risks & Constraints:** Web engines require significant memory (80–180 MB per instance) compared to dedicated video players.

#### KW-F-017: Microsoft Edge WebView2 Integration
- **Title:** WebView2 Runtime Hosting
- **Classification:** TECHNICAL-DECISION
- **Priority:** P0
- **Status:** PLANNED
- **Source:** Approved Decisions & Research Document, Section 6.4
- **Statement:** The web wallpaper player must host the Chromium-based Microsoft Edge WebView2 control embedded within a borderless Win32 window parented behind desktop icons.
- **Rationale:** Leveraging the system-installed WebView2 runtime provides cutting-edge web platform capabilities without bundling a multi-hundred-megabyte Chromium engine into the installer.
- **Acceptance Criteria:**
  1. Player initializes `CoreWebView2Controller` attached to the desktop window handle.
  2. Disables default browser UI elements (context menus, devtools shortcut keys, scrollbars, status bar).
  3. WebView2 user data folder (UDF) is isolated in a dedicated KinetiWall local cache directory.
- **Verification Method:** Integration test instantiating WebView2 controller, validating window attachment and headless/presentation flags.
- **Dependencies:** Microsoft Edge WebView2 Runtime, Win32 API.
- **Risks & Constraints:** System without WebView2 installed requires runtime detection and installation guidance (see UDR-006).

#### KW-F-018: Web Wallpaper Security Sandbox & Filesystem Isolation
- **Title:** Web Content Security Policy and Local Access Restriction
- **Classification:** USER-REQUIRED
- **Priority:** P0
- **Status:** PLANNED
- **Source:** User Project Brief, Section 6.4 & Section 7.3
- **Statement:** Interactive web wallpapers must run within a restricted security perimeter. Web wallpapers must not have unrestricted access to the local host filesystem, command execution, or arbitrary local network resources.
- **Rationale:** Third-party web wallpapers could contain malicious scripts attempting to read user documents, install malware, or access network cameras/microphones.
- **Acceptance Criteria:**
  1. Web pages are served via a virtual host mapping (e.g., `https://kinetiwall.local/`) restricted strictly to the specific wallpaper asset directory.
  2. Direct `file://` protocol access to the root filesystem is blocked.
  3. WebView2 permissions for Camera, Microphone, Geolocation, and OS Notifications are denied by default.
  4. Node.js integration and native process execution are disabled.
- **Verification Method:** Automated security penetration test attempting to read `C:\Windows\System32\drivers\etc\hosts` and access camera APIs from within a web wallpaper script; assert immediate security violation rejection.
- **Dependencies:** KW-F-017.
- **Risks & Constraints:** Some community wallpapers expect relative path access; virtual host mapping must correctly resolve local asset subfolders.

#### KW-F-019: Web Wallpaper Customization Bridge (JavaScript IPC)
- **Title:** Bi-directional JavaScript Property Bridge
- **Classification:** USER-REQUIRED
- **Priority:** P1
- **Status:** PLANNED
- **Source:** User Project Brief, Section 6.4, 6.10
- **Statement:** The web player must inject a lightweight, secure JavaScript API bridge (`window.kinetiwall`) into the web wallpaper context to allow dynamic runtime configuration (colors, speed, sliders) and audio FFT data passing from the core controller.
- **Rationale:** Allows wallpaper authors to create user-customizable widgets and reactive visuals responding to settings or music.
- **Acceptance Criteria:**
  1. `window.kinetiwall.onPropertyChange(callback)` correctly delivers property updates dispatched from the KinetiWall UI.
  2. `window.kinetiwall.onAudioData(callback)` delivers normalized audio spectrum arrays at 30–60 Hz during active playback.
- **Verification Method:** Integration test loading a mock web wallpaper, injecting property updates via `postWebMessageAsJson`, and asserting visual property changes.
- **Dependencies:** KW-F-017, KW-F-031, KW-F-038.
- **Risks & Constraints:** JSON serialization overhead of high-frequency audio data must be optimized to prevent garbage collection pauses in the web engine.

---

### 2.5 Multi-Monitor Management

#### KW-F-020: Independent Per-Display Wallpaper Assignment
- **Title:** Per-Monitor Independent Wallpaper Mode
- **Classification:** USER-REQUIRED
- **Priority:** P0
- **Status:** PLANNED
- **Source:** User Project Brief, Section 6.5
- **Statement:** KinetiWall must support assigning distinct, independent wallpapers (video, web, or static) to each active monitor in a multi-display setup.
- **Rationale:** Users frequently desire different wallpapers on secondary vertical, portrait, or tertiary displays.
- **Acceptance Criteria:**
  1. The system enumerates all active monitors via `EnumDisplayMonitors`.
  2. Each monitor can be assigned a different wallpaper asset, engine type, and volume/customization profile.
  3. Dedicated player windows or viewports are mapped to the respective monitor bounds (`rcMonitor`).
- **Verification Method:** Multi-display test fixture verifying separate player instances or viewports rendering to distinct monitor coordinates.
- **Dependencies:** Win32 Multi-Monitor API (`EnumDisplayMonitors`, `GetMonitorInfoW`).
- **Risks & Constraints:** Multiple simultaneous video players increase GPU decode session count; hardware decode limits on consumer GPUs (e.g. NVDEC session limits) must be handled.

#### KW-F-021: Multi-Display Wallpaper Spanning Mode
- **Title:** Single Wallpaper Canvas Spanning Across Multiple Displays
- **Classification:** USER-REQUIRED
- **Priority:** P1
- **Status:** PLANNED
- **Source:** User Project Brief, Section 6.5
- **Statement:** KinetiWall must support spanning a single panoramic wallpaper across the combined virtual desktop bounds of all connected monitors.
- **Rationale:** Users with dual or triple panoramic monitor setups expect ultra-wide video or canvas animations spanning their entire workspace.
- **Acceptance Criteria:**
  1. Computes total virtual desktop bounding box (`SM_XVIRTUALSCREEN`, `SM_YVIRTUALSCREEN`, `SM_CXVIRTUALSCREEN`, `SM_CYVIRTUALSCREEN`).
  2. Renders a single unified viewport cropped/scaled across the virtual screen bounds.
- **Verification Method:** Automated layout test verifying virtual screen calculation across simulated multi-monitor topologies.
- **Dependencies:** Win32 Virtual Screen Metrics.
- **Risks & Constraints:** Monitors with mismatched aspect ratios, resolutions, or DPI scaling create visual alignment discontinuities across bezel boundaries.

#### KW-F-022: Multi-Display Wallpaper Duplicate / Clone Mode
- **Title:** Synchronized Cloned Wallpaper Display
- **Classification:** USER-REQUIRED
- **Priority:** P1
- **Status:** PLANNED
- **Source:** User Project Brief, Section 6.5
- **Statement:** KinetiWall must support duplicating/cloning the same wallpaper across all connected displays, maintaining synchronized frame playback.
- **Rationale:** Provides consistent aesthetic on presentation displays, mirrored monitors, or matched multi-display setups.
- **Acceptance Criteria:**
  1. The same wallpaper asset is presented on all active displays.
  2. Video frame presentation is synchronized (or rendered via a shared texture/swapchain) without independent audio playback phase offsets.
- **Verification Method:** Multi-monitor test asserting identical playback position and single audio stream.
- **Dependencies:** KW-F-020.
- **Risks & Constraints:** Synchronizing independent decoders can drift; requires shared master clock or shared swapchain composition.

#### KW-F-023: Dynamic Display Hotplug & Topology Change Detection
- **Title:** Display Reconfiguration & Hotplug Adaptation
- **Classification:** USER-REQUIRED
- **Priority:** P0
- **Status:** PLANNED
- **Source:** User Project Brief, Section 6.5
- **Statement:** KinetiWall must dynamically handle display hotplug events (connecting or disconnecting monitors), resolution changes, orientation shifts, and monitor primary switches without requiring an application restart.
- **Rationale:** Laptops connecting/disconnecting from docking stations or external monitors are common in modern workflows.
- **Acceptance Criteria:**
  1. The core controller intercepts `WM_DISPLAYCHANGE` messages.
  2. Re-enumerates display topology within 500ms of the notification.
  3. Re-positions or destroys/creates player windows to match the new monitor count and dimensions without leaking windows.
- **Verification Method:** Simulated display change test sending `WM_DISPLAYCHANGE` and verifying window layout reconfiguration.
- **Dependencies:** Win32 `WM_DISPLAYCHANGE`, `EnumDisplayMonitors`.
- **Risks & Constraints:** Rapid docking/undocking sequences can produce bursts of multiple `WM_DISPLAYCHANGE` events; requires a 500ms debounce timer.

#### KW-F-024: Per-Monitor DPI Scaling & Mixed-DPI Support
- **Title:** Mixed-DPI and High-DPI Awareness (Per-Monitor V2)
- **Classification:** TECHNICAL-DECISION
- **Priority:** P0
- **Status:** PLANNED
- **Source:** Technical Decisions, Section 6.5 & Section 7.5
- **Statement:** All KinetiWall windows (core, gallery UI, and players) must be Per-Monitor V2 DPI aware, rendering crisp graphics across setups with mixed scaling factors (e.g., 200% laptop display alongside 100% external desktop monitor).
- **Rationale:** Mixed DPI is standard on Windows laptops. Non-DPI-aware apps appear blurry or get positioned incorrectly by Windows DWM virtualization.
- **Acceptance Criteria:**
  1. Application manifest declares `PerMonitorV2` DPI awareness.
  2. Player window geometry and swapchains adapt immediately to `WM_DPICHANGED`.
  3. No bitmap stretching or coordinate drift on multi-DPI monitor boundaries.
- **Verification Method:** DPI transition testing across simulated 100%, 150%, and 200% displays; verify window bounds match physical pixel dimensions.
- **Dependencies:** Win32 DPI API (`SetProcessDpiAwarenessContext`, `GetDpiForMonitor`).
- **Risks & Constraints:** None.

---

### 2.6 Foreground Detection & Resource Management

#### KW-F-025: Fullscreen Application & Game Detection
- **Title:** Active Fullscreen Window Detection
- **Classification:** USER-REQUIRED
- **Priority:** P0
- **Status:** PLANNED
- **Source:** User Project Brief, Section 6.6 & Section 7.1
- **Statement:** KinetiWall must detect when a demanding 3D application, PC game, or video player enters exclusive or borderless fullscreen mode on any monitor and immediately trigger configured resource-saving actions.
- **Rationale:** Live wallpapers must never steal GPU compute cycles or memory bandwidth from gaming or 3D rendering workloads.
- **Acceptance Criteria:**
  1. Detects foreground window whose bounding rectangle equals or exceeds the monitor's physical bounds (`rcMonitor`).
  2. Validates window styles (`WS_POPUP`, absence of `WS_CAPTION`) or uses `SHQueryUserNotificationState` (detecting `QUNS_RUNNING_D3D_FULL_SCREEN`).
  3. Detection latency is under 150 ms from fullscreen transition.
- **Verification Method:** Automated test launching a borderless fullscreen test window and verifying detection event logging.
- **Dependencies:** Win32 Shell API (`SHQueryUserNotificationState`, `GetForegroundWindow`).
- **Risks & Constraints:** False positives from desktop shell or taskbar overlays must be filtered.

#### KW-F-026: Discrete Resource Management States (Pause vs Suspend vs Stop)
- **Title:** Granular Wallpaper Deactivation States
- **Classification:** USER-REQUIRED
- **Priority:** P0
- **Status:** PLANNED
- **Source:** User Project Brief, Section 6.6 & Unresolved Decisions
- **Statement:** The engine must implement distinct, configurable resource management states when wallpaper rendering is suppressed:
  1. **Playback Paused:** Decoding halts; presentation paused; last frame retained in swapchain; zero decoder CPU usage.
  2. **Rendering Suspended:** Swapchain presentation halted; timers stopped; GPU draw calls halted.
  3. **Player Stopped:** Media stream closed or player process suspended/terminated; GPU VRAM buffers released.
  4. **Static Fallback Active:** Active video/web player replaced with a lightweight static image.
- **Rationale:** Different user hardware demands different tradeoffs between resume latency and memory/GPU resource freeing.
- **Acceptance Criteria:**
  1. Entering "Playback Paused" reduces player CPU utilization to 0.0% within 100 ms.
  2. Entering "Player Stopped" releases allocated GPU video surfaces and decoder contexts.
  3. Resuming from "Playback Paused" takes less than 50 ms.
- **Verification Method:** Performance profiling verifying CPU/GPU thread states and VRAM allocations across state transitions.
- **Dependencies:** KW-F-014, KW-F-025.
- **Risks & Constraints:** Re-initializing a fully stopped player process incurs higher latency (~200–500ms) than resuming a paused swapchain.

#### KW-F-027: Maximized Window & Desktop Occlusion Detection
- **Title:** Occlusion and Maximized Window State Detection
- **Classification:** TECHNICAL-DECISION
- **Priority:** P1
- **Status:** PLANNED
- **Source:** Approved Decisions & Section 6.6
- **Statement:** The engine must detect when a monitor's desktop surface is fully occluded by maximized or overlapping windows and optionally pause wallpaper rendering on that specific monitor.
- **Rationale:** If the user has maximized a browser or IDE covering the entire monitor, wallpaper rendering is invisible and needlessly consumes power.
- **Acceptance Criteria:**
  1. Checks if the foreground window on a target monitor is maximized (`IsZoomed`) and completely covers the desktop work area (`rcWork`).
  2. Pauses rendering on that monitor while leaving un-occluded monitors active.
- **Verification Method:** Occlusion test maximizing a window on Monitor 1 while leaving Monitor 2 clear; assert Monitor 1 pauses and Monitor 2 continues rendering.
- **Dependencies:** KW-F-020, KW-F-026.
- **Risks & Constraints:** Transparent or translucent windows (Mica/Acrylic) may reveal the wallpaper; occlusion policy must offer configurable strictness.

#### KW-F-028: Low-Overhead Window Monitoring Mechanism
- **Title:** Event-Driven WinEventHook Foreground Tracking
- **Classification:** TECHNICAL-DECISION
- **Priority:** P0
- **Status:** PLANNED
- **Source:** Research Document, Section 6.6 & Section 7.1
- **Statement:** Foreground and window state tracking must use asynchronous event hooks (`SetWinEventHook` listening for `EVENT_SYSTEM_FOREGROUND` and `EVENT_OBJECT_LOCATIONCHANGE`) rather than tight high-frequency polling loops.
- **Rationale:** High-frequency polling (e.g., calling `GetForegroundWindow` every 10ms) creates constant background CPU wakeups, degrading laptop battery life.
- **Acceptance Criteria:**
  1. Foreground transitions are captured via OS event callbacks.
  2. Background controller CPU utilization during idle desktop stays below 0.1%.
- **Verification Method:** CPU profiler inspection during 10-minute idle desktop test; assert absence of continuous polling wakeups.
- **Dependencies:** Win32 `SetWinEventHook`, `UnhookWinEvent`.
- **Risks & Constraints:** Event hooks can occasionally miss transitions during shell lock/unlock; a low-frequency 2000ms safety watchdog check handles edge cases.

---

### 2.7 Laptop Power Management

#### KW-F-029: Battery Mode Detection & Action
- **Title:** Dynamic AC / Battery Power State Management
- **Classification:** USER-REQUIRED
- **Priority:** P0
- **Status:** PLANNED
- **Source:** User Project Brief, Section 6.7
- **Statement:** KinetiWall must detect when the host machine transitions from AC power to battery power and execute a user-configured power-saving policy (e.g., pause live wallpapers, reduce target framerate to 30 FPS, or switch to static image).
- **Rationale:** Maximizes laptop battery endurance when untethered from wall power.
- **Acceptance Criteria:**
  1. Intercepts `WM_POWERBROADCAST` (`PBT_APMPOWERSTATUSCHANGE`) and queries `GetSystemPowerStatus`.
  2. Detects battery operation (`ACLineStatus == 0`) and triggers configured policy within 1.0 second.
  3. Restores standard performance profile immediately when AC power is reconnected.
- **Verification Method:** Battery simulation test triggering power status broadcasts; assert policy execution and recovery.
- **Dependencies:** Win32 Power Management API.
- **Risks & Constraints:** None.

#### KW-F-030: Windows Battery Saver Mode Integration
- **Title:** Windows Battery Saver Mode Adaptation
- **Classification:** TECHNICAL-DECISION
- **Priority:** P1
- **Status:** PLANNED
- **Source:** Technical Decisions, Section 6.7
- **Statement:** KinetiWall must detect Windows Battery Saver mode activation (`SystemStatusFlag == 1`) and automatically enforce the highest resource conservation profile (pausing all rendering engines).
- **Rationale:** Aligns KinetiWall with the operating system's explicit low-power state.
- **Acceptance Criteria:**
  1. Detects Windows Battery Saver state changes via power status query.
  2. Immediately pauses or stops all active wallpaper rendering.
- **Verification Method:** Automated test toggling Battery Saver state and validating suspension of all player processes.
- **Dependencies:** KW-F-029.
- **Risks & Constraints:** None.

---

### 2.8 Audio Visualization Engine

#### KW-F-031: WASAPI Audio Loopback Capture
- **Title:** Real-Time Loopback Audio Capture
- **Classification:** USER-REQUIRED
- **Priority:** P1
- **Status:** PLANNED
- **Source:** User Project Brief, Section 6.8
- **Statement:** KinetiWall must provide an opt-in audio visualization subsystem capturing system audio output using the Windows Audio Session API (WASAPI) in loopback mode (`AUDCLNT_STREAMFLAGS_LOOPBACK`).
- **Rationale:** Audio-reactive wallpapers require real-time system audio data to drive visual effects in sync with music or media.
- **Acceptance Criteria:**
  1. Captures system audio from the active default multimedia render endpoint.
  2. Audio capture is completely opt-in and activates **only** when an active wallpaper requests audio reactivity.
  3. When no audio-reactive wallpaper is running, the WASAPI capture thread is fully terminated.
- **Verification Method:** Audio test playing test audio tone and asserting captured sample buffers in memory.
- **Dependencies:** Windows WASAPI (`CoreAudio` APIs, `IMMDeviceEnumerator`).
- **Risks & Constraints:** Requires audio endpoint re-initialization when the user changes headphones or default audio devices.

#### KW-F-032: Real-Time Audio Frequency Spectrum Analysis (FFT)
- **Title:** In-Memory Fast Fourier Transform (FFT) Audio Processing
- **Classification:** USER-REQUIRED
- **Priority:** P1
- **Status:** PLANNED
- **Source:** User Project Brief, Section 6.8
- **Statement:** Captured PCM audio samples must be processed in real-time using a Fast Fourier Transform (FFT) algorithm into normalized frequency bands (bass, mid, treble, or 16–128 spectrum bins) and delivered to active player engines.
- **Rationale:** Wallpapers need pre-computed frequency magnitude arrays rather than raw audio streams.
- **Acceptance Criteria:**
  1. FFT window calculation (e.g., Hann window, 1024/2048 samples) runs on a dedicated background thread.
  2. Spectrum analysis data is delivered via IPC or shared memory at 30–60 updates per second.
  3. Processing CPU overhead remains below 0.5% CPU.
- **Verification Method:** Automated DSP test feeding known sine waves (e.g. 100 Hz, 1 kHz, 10 kHz) and asserting dominant peaks in corresponding FFT output bins.
- **Dependencies:** KW-F-031, Rust DSP/FFT crate (e.g., `rustfft`).
- **Risks & Constraints:** High-frequency IPC data transfer must avoid memory copies or heap fragmentation.

#### KW-F-033: Audio Privacy, Idle Silence & Device Change Handling
- **Title:** Audio Privacy Guarantee & Dynamic Endpoint Switching
- **Classification:** USER-REQUIRED
- **Priority:** P0
- **Status:** PLANNED
- **Source:** User Project Brief, Section 6.8, Section 7.4 & KW-NG-005
- **Statement:** The audio engine must process audio strictly in volatile memory. It must never save, record, persist to disk, or transmit audio data over any network. When system audio is silent, the capture thread must enter an idle sleep state. When default audio devices change, it must automatically re-bind.
- **Rationale:** Protects user privacy against eavesdropping concerns and prevents resource waste during silence.
- **Acceptance Criteria:**
  1. Zero audio recording code, disk write buffers, or network transmission routines exist in the audio module.
  2. If system audio drops to silence (RMS < -60 dB) for >3 seconds, FFT processing sleeps until audio returns.
  3. Device changes (`IMMNotificationClient::OnDefaultDeviceChanged`) re-initialize the loopback stream seamlessly within 500 ms.
- **Verification Method:** Security/privacy audit verifying absence of audio file creation or network sockets; device unplug/plug test asserting audio recovery.
- **Dependencies:** KW-F-031, KW-F-032.
- **Risks & Constraints:** None.

---

### 2.9 Wallpaper Library & Asset Management

#### KW-F-034: Local Wallpaper Import & Packaging
- **Title:** Local Media & Package Import Subsystem
- **Classification:** USER-REQUIRED
- **Priority:** P0
- **Status:** PLANNED
- **Source:** User Project Brief, Section 6.9
- **Statement:** The application must allow users to import local media files (MP4, WebM, AVI) and interactive web wallpaper directories/packages into an organized local library.
- **Rationale:** Users need a simple, reliable way to add their own video clips and web animations to their wallpaper catalog.
- **Acceptance Criteria:**
  1. Supports drag-and-drop file import into the gallery UI and file dialog selection.
  2. Validates imported media container and structure before adding to library.
  3. Copies or links assets into KinetiWall's local library folder (`%LOCALAPPDATA%\KinetiWall\wallpapers`).
- **Verification Method:** UI and CLI import tests loading diverse file formats and validating catalog entries.
- **Dependencies:** Filesystem APIs, KW-F-010, KW-F-016.
- **Risks & Constraints:** Large video files (>2 GB) should be linked rather than duplicated to avoid exhausting user disk space.

#### KW-F-035: Metadata Indexing & Database Management
- **Title:** Local Wallpaper Metadata Store (SQLite / JSON)
- **Classification:** TECHNICAL-DECISION
- **Priority:** P1
- **Status:** PLANNED
- **Source:** Approved Decisions & Section 6.9
- **Statement:** Wallpaper metadata (title, author, description, tags, file path, resolution, duration, properties schema, favorites) must be indexed in a structured, local, queryable store (e.g., SQLite database or indexed JSON catalog).
- **Rationale:** Provides fast search, tagging, filtering, and sorting across thousands of wallpapers without re-scanning disk directories.
- **Acceptance Criteria:**
  1. Metadata queries (search by name, filter by tag, sort by date) execute in under 20 ms for catalogs up to 10,000 items.
  2. Schema supports versioning and automatic database migrations.
  3. Fully functional without network connectivity.
- **Verification Method:** Performance test benchmarking search and query response times against a populated database of 5,000 items.
- **Dependencies:** Embedded SQLite (`rusqlite`) or structured JSON store.
- **Risks & Constraints:** Database corruption recovery must be implemented (automatic backup/rebuild from asset files).

#### KW-F-036: Automated Thumbnail Generation & Caching
- **Title:** Background Thumbnail Extractor and Cache
- **Classification:** USER-REQUIRED
- **Priority:** P1
- **Status:** PLANNED
- **Source:** User Project Brief, Section 6.9
- **Statement:** When new wallpapers are imported, KinetiWall must automatically generate preview thumbnails (extracting representative video frames or rendering web snapshots) and store them in an optimized local thumbnail cache.
- **Rationale:** Fast gallery scrolling requires pre-computed thumbnail images rather than on-the-fly video decoding.
- **Acceptance Criteria:**
  1. Extracts a crisp video frame (e.g. at 10% timestamp) and saves as WebP/JPEG thumbnail.
  2. Thumbnail generation runs asynchronously in background without freezing the UI.
  3. Cached thumbnails load into the gallery UI with zero hitching.
- **Verification Method:** Automated test importing 20 video files; verify 20 corresponding valid thumbnail files are written to cache within 5.0 seconds.
- **Dependencies:** KW-F-010, image encoding crate.
- **Risks & Constraints:** High-resolution videos may take several seconds to extract; queue management must prioritize visible viewport items.

#### KW-F-037: Library Search, Filtering & Favorites
- **Title:** Library Catalog Organization & Search
- **Classification:** USER-REQUIRED
- **Priority:** P1
- **Status:** PLANNED
- **Source:** User Project Brief, Section 6.9
- **Statement:** The gallery UI must provide real-time search by title/tag, filtering by wallpaper type (Video, Web, Interactive), sorting (Name, Date, Size), and a toggleable "Favorites" flag.
- **Rationale:** Ensures rapid navigation and organization of growing wallpaper collections.
- **Acceptance Criteria:**
  1. Search input updates the display list within 50 ms of keystroke (debounced).
  2. Filter toggles instantly isolate video or web wallpapers.
  3. Favorited items persist across application restarts.
- **Verification Method:** UI functional test executing search queries and filter combinations against populated test data.
- **Dependencies:** KW-F-035.
- **Risks & Constraints:** None.

---

### 2.10 Customization & Property Bridge

#### KW-F-038: Dynamic Wallpaper Property Customization
- **Title:** Runtime Property Adjustment
- **Classification:** USER-REQUIRED
- **Priority:** P1
- **Status:** PLANNED
- **Source:** User Project Brief, Section 6.10
- **Statement:** KinetiWall must support live, runtime customization of wallpaper properties, including playback speed, brightness, contrast, saturation, audio volume, and custom wallpaper-defined sliders, color pickers, and boolean switches.
- **Rationale:** Users want to personalize their wallpapers (e.g., match room lighting, tune animation speed, change theme colors).
- **Acceptance Criteria:**
  1. Changing a slider in the UI updates the running wallpaper within 30 ms without reloading or restarting the player process.
  2. Color picker values update web shader uniforms or CSS custom properties in real-time.
- **Verification Method:** Integration test verifying IPC property update dispatch and receipt by running player.
- **Dependencies:** KW-F-008, KW-F-014, KW-F-019.
- **Risks & Constraints:** Video color adjustments require hardware shader support or libmpv video filters (`eq`).

#### KW-F-039: Property Schema Definition & Validation
- **Title:** Structured Manifest Property Schema & Clamping
- **Classification:** TECHNICAL-DECISION
- **Priority:** P1
- **Status:** PLANNED
- **Source:** Technical Decisions, Section 6.10
- **Statement:** Custom properties exposed by interactive or video wallpapers must be declared in a structured JSON manifest (`kinetiwall.json` or `project.json`) and strictly validated, clamped to min/max ranges, and sanitized before application.
- **Rationale:** Prevents invalid property values (e.g., negative speed, `NaN`, out-of-bound colors) from crashing rendering engines or corrupting configurations.
- **Acceptance Criteria:**
  1. Parser strictly validates property types (`integer`, `float`, `color`, `boolean`, `select`).
  2. Out-of-range numeric inputs are clamped to defined boundaries.
  3. Malformed schemas fail gracefully with clear diagnostic warnings.
- **Verification Method:** Unit tests exercising schema parser with valid, out-of-range, and malformed JSON payloads.
- **Dependencies:** `serde` JSON parser.
- **Risks & Constraints:** Backwards compatibility with third-party wallpaper manifest formats (e.g., Lively/LivelyInfo, Wallpaper Engine `project.json`) requires schema translation layers.

#### KW-F-040: Configuration Persistence & Versioning
- **Title:** Reliable Configuration Persistence
- **Classification:** USER-REQUIRED
- **Priority:** P0
- **Status:** PLANNED
- **Source:** User Project Brief, Section 6.10
- **Statement:** User settings, per-monitor wallpaper assignments, and customized properties must persist atomically to disk in a human-readable, versioned format (`settings.json`), with backup protection against power loss or crashes.
- **Rationale:** Settings must never be lost due to abrupt system shutdown.
- **Acceptance Criteria:**
  1. Settings are written via atomic file replacement (`write to temp` -> `FlushFileBuffers` -> `MoveFileExW` with `REPLACE_EXISTING`).
  2. File format contains a `schema_version` integer.
  3. Automatic migration runs when newer versions introduce schema changes.
- **Verification Method:** Automated test writing settings and simulating sudden process kill; assert configuration integrity upon reload.
- **Dependencies:** Windows Filesystem API.
- **Risks & Constraints:** None.

---

### 2.11 System Tray & Application Lifecycle

#### KW-F-041: System Tray Presence & Quick Actions
- **Title:** Native Windows Notification Area (System Tray) Integration
- **Classification:** USER-REQUIRED
- **Priority:** P0
- **Status:** PLANNED
- **Source:** User Project Brief, Section 6.11
- **Statement:** KinetiWall must display an icon in the Windows notification area (system tray) with a responsive right-click context menu offering instant controls: Open Gallery, Pause/Resume All, Next Wallpaper, Mute Audio, Display Settings, and Exit.
- **Rationale:** Provides unintrusive, omnipresent control over the desktop wallpaper without cluttering the taskbar.
- **Acceptance Criteria:**
  1. System tray icon registers via `Shell_NotifyIconW`.
  2. Right-clicking invokes a native Windows popup menu within 50 ms.
  3. Left-clicking or double-clicking launches/focuses the gallery UI.
  4. Tray icon handles `TaskbarCreated` message to restore icon if Explorer restarts.
- **Verification Method:** Win32 message test asserting tray icon registration and menu command dispatch.
- **Dependencies:** Win32 Shell API (`Shell_NotifyIconW`, `TrackPopupMenuEx`).
- **Risks & Constraints:** DPI scaling of tray icon assets across mixed-DPI displays.

#### KW-F-042: Windows Startup (Run on Boot) Integration
- **Title:** Optional Launch on Windows Startup
- **Classification:** USER-REQUIRED
- **Priority:** P1
- **Status:** PLANNED
- **Source:** User Project Brief, Section 6.11
- **Statement:** KinetiWall must offer an optional user setting to launch automatically when the user logs into Windows, registering via the standard Windows `HKCU\Software\Microsoft\Windows\CurrentVersion\Run` registry key or a Windows Startup shortcut.
- **Rationale:** Live wallpapers are expected to be active automatically upon desktop login.
- **Acceptance Criteria:**
  1. Toggling "Run on Startup" creates/removes the appropriate registry entry pointing to `kinetiwall-core.exe --autostart`.
  2. Startup launch boots KinetiWall silently into the system tray without popping up the gallery UI.
- **Verification Method:** Integration test toggling the setting, asserting registry state, and validating command line parameters.
- **Dependencies:** Windows Registry API (`RegOpenKeyExW`, `RegSetValueExW`).
- **Risks & Constraints:** Antivirus software occasionally monitors `Run` key writes; clear user intent must accompany the action.

#### KW-F-043: Single-Instance Enforcement & Inter-Process Activation
- **Title:** Single-Instance Application Mutex
- **Classification:** TECHNICAL-DECISION
- **Priority:** P0
- **Status:** PLANNED
- **Source:** Technical Decisions, Section 6.11
- **Statement:** KinetiWall must enforce a strict single-instance policy per user session using a named Windows Mutex. Attempting to launch a second instance must signal the existing instance to open its gallery UI and exit immediately.
- **Rationale:** Prevents duplicate daemon processes, duplicate tray icons, and conflicting window attachments.
- **Acceptance Criteria:**
  1. Core controller creates a named session mutex (`Local\KinetiWall_Core_SingleInstance_Mutex`).
  2. Second launch attempt detects existing mutex, dispatches an IPC message to open the gallery, and terminates cleanly with exit code 0 within 200 ms.
- **Verification Method:** Launch collision test executing two simultaneous instances; assert only one core process remains active.
- **Dependencies:** Win32 `CreateMutexW`.
- **Risks & Constraints:** Mutex abandonment must be handled cleanly.

---

### 2.12 Native 3D & Executable Integration

#### KW-F-044: Native 3D / Game Engine Window Hosting Evaluation
- **Title:** Evaluation of Native Standalone 3D Executable Integration
- **Classification:** RESEARCH-RECOMMENDATION
- **Priority:** P3
- **Status:** UNRESOLVED
- **Source:** Research Document, Section 6.12 & UDR-011
- **Statement:** Evaluate the feasibility, security boundaries, and lifecycle management of hosting standalone native 3D executables (DirectX, OpenGL, Vulkan, Unity, Godot exports) as desktop wallpapers via `SetParent` window parenting.
- **Rationale:** Proposed in competitor benchmarks (e.g. Wallpaper Engine executable wallpapers) to enable high-performance 3D scenes.
- **Acceptance Criteria:**
  1. Feasibility study detailing window parenting risks, GPU handle leaks, child process sandboxing, and crash isolation.
  2. Prototype validation or formal deferral recommendation document produced.
- **Verification Method:** Technical spike and architectural review evaluating `SetParent` vs out-of-process IPC rendering.
- **Dependencies:** Win32 Window Parenting API.
- **Risks & Constraints:** Running arbitrary unsigned executables poses severe security, stability, and anti-cheat risks. Must remain classified as unapproved research until explicitly authorized.

---

### 2.13 Offline Operation & Dependency Packaging

#### KW-F-045: 100% Offline Core Functionality Guarantee
- **Title:** Full Local Offline Operability
- **Classification:** USER-REQUIRED
- **Priority:** P0
- **Status:** PLANNED
- **Source:** User Project Brief, Section 6.13 & Section 7.4
- **Statement:** All core wallpaper playback, library management, property customization, multi-monitor configuration, and audio visualization must function completely without an active internet connection.
- **Rationale:** Users must never be prevented from enjoying local desktop customization due to internet outages or offline environments.
- **Acceptance Criteria:**
  1. Disconnecting network interfaces (`netsh interface set interface ... admin=disable`) leaves all KinetiWall functionality 100% operational.
  2. No blocking network timeouts, modal warnings, or failed network requests impede startup or playback.
- **Verification Method:** Offline test suite executed with all network adapters disabled; assert complete test pass.
- **Dependencies:** Local asset storage, local database.
- **Risks & Constraints:** None.

#### KW-F-046: Self-Contained Dependency Distribution Strategy
- **Title:** Offline Runtime Packaging & Bootstrapping
- **Classification:** USER-REQUIRED
- **Priority:** P0
- **Status:** PLANNED
- **Source:** User Project Brief, Section 6.13, Section 7.8 & UDR-006
- **Statement:** All required runtime libraries (media decoders, C/C++ runtimes, Direct3D shaders) must be bundled within the offline installer or rely strictly on components guaranteed by modern Windows installations.
- **Rationale:** Eliminates failed installations caused by missing online downloaders or firewalled environments.
- **Acceptance Criteria:**
  1. Installer executes and installs successfully on an air-gapped Windows machine.
  2. Clear handling for WebView2: detects system-installed Evergreen runtime, or provides an offline installer bundle.
- **Verification Method:** Clean installation test on an isolated offline Windows 10/11 virtual machine.
- **Dependencies:** Installer build tooling (NSIS / Inno Setup / WiX).
- **Risks & Constraints:** Bundling fixed-version WebView2 increases installer size by ~150 MB (see UDR-006).

---

## 3. Non-Functional Requirements Inventory (KW-NF)

### 3.1 Performance & Resource Efficiency

#### KW-NF-001: Idle Core Controller Memory Footprint
- **Title:** Ultra-Low Idle Daemon RAM Usage
- **Classification:** TECHNICAL-DECISION
- **Priority:** P0
- **Status:** PROVISIONAL
- **Source:** User Project Brief & Research Document, Section 7.1
- **Statement:** The core controller daemon process (`kinetiwall-core.exe`), when idle with UI closed and supervising active players, must maintain a private working set memory footprint below **15 MB** (provisional target: < 10 MB).
- **Rationale:** Live wallpaper engines must remain unnoticeable background citizens that never crowd out demanding user workloads.
- **Acceptance Criteria:**
  1. Measured Private Working Set (RAM) of `kinetiwall-core.exe` remains ≤ 15.0 MB during continuous 1-hour idle monitoring.
- **Verification Method:** Automated performance benchmark sampling `Process.WorkingSet64` via PowerShell/perfmon over 60 minutes.
- **Dependencies:** Pure Rust implementation, minimal static allocations.
- **Risks & Constraints:** Win32 API handles and IPC pipe buffers must be carefully managed to avoid heap creep.

#### KW-NF-002: Idle Core Controller CPU Overhead
- **Title:** Sub-0.1% Idle CPU Consumption
- **Classification:** TECHNICAL-DECISION
- **Priority:** P0
- **Status:** PROVISIONAL
- **Source:** User Project Brief & Research Document, Section 7.1
- **Statement:** The core controller process, when not actively handling IPC commands or window transitions, must consume less than **0.1%** average CPU utilization on a standard multi-core desktop processor.
- **Rationale:** Background daemons must not cause CPU core frequency scaling or battery consumption.
- **Acceptance Criteria:**
  1. Average CPU usage over a 10-minute window with no user interaction measures < 0.10%.
  2. Thread wait states rely on event-driven blocking primitives (`GetMessageW`, `WaitForMultipleObjects`), never busy loops.
- **Verification Method:** Windows Performance Toolkit (WPT) trace capturing CPU thread execution states during idle desktop.
- **Dependencies:** KW-F-028.
- **Risks & Constraints:** Timer resolution on Windows must not be globally forced to 1ms unless actively needed.

#### KW-NF-003: Video Player CPU Overhead under Active Playback
- **Title:** Minimal Hardware-Accelerated Video CPU Utilization
- **Classification:** USER-REQUIRED
- **Priority:** P0
- **Status:** PROVISIONAL
- **Source:** User Project Brief & Research Document, Section 7.1
- **Statement:** Active hardware-accelerated playback of a 1080p 60 FPS H.264 video wallpaper must consume less than **3.0%** aggregate CPU utilization on reference hardware (Quad-Core Intel i5 / AMD Ryzen 5 or equivalent).
- **Rationale:** Live video wallpapers must not tax the CPU; decoding and presentation must be offloaded to dedicated GPU fixed-function hardware.
- **Acceptance Criteria:**
  1. 1080p60 H.264 playback maintains average CPU utilization ≤ 3.0% across 5 minutes of continuous playback.
  2. Dedicated GPU 3D engine utilization remains ≤ 5.0% (excluding dedicated video decode engine NVDEC/VCN).
- **Verification Method:** Hardware performance benchmark recording CPU and GPU engine counters using Windows Performance Counters (`\Processor(_Total)\% Processor Time`).
- **Dependencies:** KW-F-012.
- **Risks & Constraints:** High-bitrate 4K VP9/AV1 videos may require slightly higher CPU allocations; targets are calibrated to 1080p60 baseline.

#### KW-NF-004: Rapid Resource Suspension & Pause Latency
- **Title:** Sub-100ms Suspension Response Latency
- **Classification:** TECHNICAL-DECISION
- **Priority:** P0
- **Status:** PROVISIONAL
- **Source:** User Project Brief & Research Document, Section 7.1
- **Statement:** Upon detecting a fullscreen application or receiving a pause signal, active wallpaper players must halt rendering and suspend decoding within **100 ms** (provisional target: < 50 ms).
- **Rationale:** Immediate pause prevents framerate stutter during game launch or benchmark initialization.
- **Acceptance Criteria:**
  1. Time elapsed between the trigger event (`EVENT_SYSTEM_FOREGROUND` or IPC command) and the last presented video swapchain frame is ≤ 100 ms.
  2. Player CPU utilization drops to 0.0% within 150 ms of the trigger event.
- **Verification Method:** High-precision timestamped event logging capturing trigger dispatch time vs decoder pause confirmation timestamp.
- **Dependencies:** KW-F-025, KW-F-026.
- **Risks & Constraints:** Pipe messaging latency and decoder thread synchronization must be non-blocking.

#### KW-NF-005: Zero Gaming Frame-Time Impact When Paused
- **Title:** Transparent Gaming Performance Parity
- **Classification:** USER-REQUIRED
- **Priority:** P0
- **Status:** PLANNED
- **Source:** User Project Brief, Section 7.1
- **Statement:** When a game or 3D graphics benchmark is running in fullscreen mode, KinetiWall must introduce **0.0%** measurable degradation in 99th percentile frame times (1% lows) or average FPS compared to a clean system without KinetiWall installed.
- **Rationale:** Gamers will unreservedly uninstall any customization utility that degrades gaming smoothness.
- **Acceptance Criteria:**
  1. Automated 3D benchmark (e.g., Unigine Superposition / 3DMark or standard D3D11 render test) exhibits statistically indistinguishable FPS (within ±0.5% margin of error) with KinetiWall paused vs KinetiWall not running.
- **Verification Method:** Comparative benchmark suite running 10 iterations of a graphics test with KinetiWall active/paused vs uninstalled baseline.
- **Dependencies:** KW-F-025, KW-F-026.
- **Risks & Constraints:** Requires verification across both NVIDIA and AMD driver architectures.

#### KW-NF-006: Lightweight Static Installer Size
- **Title:** Compact Standalone Installer Target
- **Classification:** RESEARCH-RECOMMENDATION
- **Priority:** P2
- **Status:** PROVISIONAL
- **Source:** Research Document, Section 7.8 & UDR-006
- **Statement:** The core standalone installer package should achieve a compressed distribution size under **35 MB** (provisional target: < 25 MB) when leveraging system WebView2 runtime.
- **Rationale:** Compact downloads increase user adoption, reduce bandwidth overhead, and reflect software quality.
- **Acceptance Criteria:**
  1. Production release installer `.exe` size measures ≤ 35.0 MB.
- **Verification Method:** Build artifact inspection measuring installer binary byte length in release CI pipeline.
- **Dependencies:** Rust release profile optimizations (`opt-level = "z"`, `lto = true`, `codegen-units = 1`, `strip = true`).
- **Risks & Constraints:** Bundling full static ffmpeg/libmpv libraries or an offline WebView2 bootstrapper will exceed 35 MB; requires careful dependency pruning.

---

### 3.2 Reliability, Recovery & Resilience

#### KW-NF-007: Mean Time Between Failures & Uninterrupted Longevity
- **Title:** 72-Hour Continuous Playback Stability
- **Classification:** USER-REQUIRED
- **Priority:** P0
- **Status:** PLANNED
- **Source:** User Project Brief, Section 7.2
- **Statement:** The KinetiWall engine must be capable of running continuously for at least **72 hours** of uninterrupted live wallpaper playback without crashing, freezing, leaking memory, or desynchronizing desktop layers.
- **Rationale:** Desktop live wallpapers are expected to run indefinitely as part of the operating system environment.
- **Acceptance Criteria:**
  1. Zero unhandled exceptions or crashes across a 72-hour automated burn-in test.
  2. Memory growth (Working Set) over the 72-hour period does not exceed 10% of initial steady-state allocation (zero runaway leaks).
- **Verification Method:** 72-hour continuous stress test logging memory, handle counts, and playback state every 5 minutes.
- **Dependencies:** KW-F-005, KW-F-006, KW-F-010.
- **Risks & Constraints:** Third-party video decoders must be audited for unmanaged memory leaks.

#### KW-NF-008: Sleep, Modern Standby & Wake Recovery
- **Title:** System Sleep and Hibernate State Resilience
- **Classification:** USER-REQUIRED
- **Priority:** P0
- **Status:** PLANNED
- **Source:** User Project Brief, Section 7.2
- **Statement:** KinetiWall must handle system suspend, sleep (S3), modern standby (S0ix), and hibernation (S4) events gracefully, releasing graphics contexts upon sleep and re-initializing render pipelines cleanly upon wake.
- **Rationale:** Laptop sleep/wake cycles frequently reset Direct3D device contexts (`DXGI_ERROR_DEVICE_RESET` / `DXGI_ERROR_DEVICE_REMOVED`), causing unhandled crashes in poorly architected graphics utilities.
- **Acceptance Criteria:**
  1. Intercepts `WM_POWERBROADCAST` (`PBT_APMSUSPEND` and `PBT_APMRESUMEAUTOMATIC`).
  2. Successfully restores live wallpaper rendering within 2.0 seconds of system wake without requiring user intervention.
- **Verification Method:** Automated sleep/wake cycle testing via Windows Power Management test scripts (`pwrtest.exe`).
- **Dependencies:** Win32 Power Management, Direct3D Device Recovery.
- **Risks & Constraints:** Device removed errors require rebuilding swapchains from scratch.

#### KW-NF-009: Corrupt Media Resilience & Fault Isolation
- **Title:** Resilient Handling of Truncated or Malformed Media
- **Classification:** USER-REQUIRED
- **Priority:** P1
- **Status:** PLANNED
- **Source:** User Project Brief, Section 7.2
- **Statement:** Attempting to play a corrupted, truncated, or invalid video file or web page must fail safely without crashing the player process, hanging the core daemon, or freezing the Windows shell.
- **Rationale:** Users will encounter corrupted downloads or unsupported encodings.
- **Acceptance Criteria:**
  1. Media parser detects invalid header or premature EOF.
  2. Player transitions to safe error state, logs the error, and signals the core controller.
  3. Core controller displays a user-friendly error pill/notification in the UI.
- **Verification Method:** Fuzz testing injecting corrupted, zero-byte, and truncated MP4/WebM files; assert zero process crashes.
- **Dependencies:** KW-F-010, KW-F-015.
- **Risks & Constraints:** None.

#### KW-NF-010: Bounded Resource Quotas & Process Sandboxing
- **Title:** Windows Job Object Process Guardrails
- **Classification:** TECHNICAL-DECISION
- **Priority:** P1
- **Status:** PLANNED
- **Source:** Technical Decisions, Section 7.2 & Section 7.3
- **Statement:** All spawned wallpaper player processes must be assigned to a Windows Job Object configured with process limits (automatic child process termination on controller exit via `JOB_OBJECT_LIMIT_KILL_ON_JOB_CLOSE`) to prevent orphaned zombie processes.
- **Rationale:** Guarantees that if the core controller terminates unexpectedly, all rendering subprocesses are immediately cleaned up by the Windows kernel.
- **Acceptance Criteria:**
  1. Job Object created with `JOB_OBJECT_LIMIT_KILL_ON_JOB_CLOSE`.
  2. Killing `kinetiwall-core.exe` via Task Manager instantly terminates all child player processes.
- **Verification Method:** Kill test terminating parent process and validating child process tree destruction via PowerShell.
- **Dependencies:** Windows Job Object API (`CreateJobObjectW`, `AssignProcessToJobObject`).
- **Risks & Constraints:** Windows 10 supports nested jobs; compatibility must be verified with WebView2 child processes.

---

### 3.3 Security & Sandboxing

#### KW-NF-011: Strict Least-Privilege Execution
- **Title:** Non-Elevated Standard User Execution
- **Classification:** USER-REQUIRED
- **Priority:** P0
- **Status:** PLANNED
- **Source:** User Project Brief, Section 7.3
- **Statement:** KinetiWall must execute entirely within a standard user security context (medium integrity level) and must **never** require Administrator privileges (UAC elevation) for standard operation, wallpaper playback, or startup configuration.
- **Rationale:** Running desktop customization tools as Administrator exposes the entire host system to privilege escalation attacks and violates enterprise security guidelines.
- **Acceptance Criteria:**
  1. Application manifest declares `requestedExecutionLevel level="asInvoker"`.
  2. All operations (registry settings in `HKCU`, files in `%LOCALAPPDATA%`, named pipes) succeed under standard non-admin accounts.
- **Verification Method:** Security verification executing KinetiWall inside a restricted standard user account; assert complete operational success with zero UAC prompts.
- **Dependencies:** Windows User Account Control (UAC).
- **Risks & Constraints:** None.

#### KW-NF-012: Named Pipe Access Control & Security Descriptors
- **Title:** Secure Single-User IPC Access Restriction
- **Classification:** TECHNICAL-DECISION
- **Priority:** P0
- **Status:** PLANNED
- **Source:** Technical Decisions, Section 7.3
- **Statement:** All IPC Named Pipes created by KinetiWall must have discretionary access control lists (DACL) restricting connection rights exclusively to the current user's security identifier (SID) and Local System, blocking unauthorized cross-session or unprivileged local user tampering.
- **Rationale:** Open named pipes allow unprivileged local processes or malware in other user sessions to inject malicious commands or hijack application execution.
- **Acceptance Criteria:**
  1. Pipe security descriptor is created using explicit SID-restricted DACL (`ConvertStringSecurityDescriptorToSecurityDescriptorW`).
  2. Connection attempts from other local accounts are rejected by the kernel with `ERROR_ACCESS_DENIED`.
- **Verification Method:** Automated security test running an unprivileged secondary test account attempting to connect to KinetiWall's named pipe; assert access denied.
- **Dependencies:** Windows Security API (`Advapi32.dll`).
- **Risks & Constraints:** None.

#### KW-NF-013: IPC Payload Strict Schema Validation & Sanitization
- **Title:** Strict Serialization and Boundary Validation
- **Classification:** TECHNICAL-DECISION
- **Priority:** P0
- **Status:** PLANNED
- **Source:** Technical Decisions, Section 7.3
- **Statement:** Every IPC message received by the core controller or player processes must be strictly validated against fixed schemas. Oversized payloads (>64 KB for control messages, >1 MB for audio arrays) must be rejected immediately.
- **Rationale:** Protects against buffer overflows, denial-of-service memory exhaustion, and command injection attacks.
- **Acceptance Criteria:**
  1. Deserialization strictly rejects unknown fields, unexpected data types, or payloads exceeding max byte limits.
  2. Buffer parsing errors trigger an immediate connection reset without crashing the host process.
- **Verification Method:** Fuzzing test sending malformed, fragmented, and oversized byte sequences to the IPC listener; assert graceful rejection.
- **Dependencies:** `serde` deserializer with strict size limits.
- **Risks & Constraints:** None.

#### KW-NF-014: Archive Extraction & Path Traversal Guard (Zip Slip Protection)
- **Title:** Secure Archive Import Validation
- **Classification:** TECHNICAL-DECISION
- **Priority:** P0
- **Status:** PLANNED
- **Source:** Technical Decisions, Section 7.3
- **Statement:** When unpacking wallpaper packages (`.zip` or custom archives), the extraction routine must validate all file paths, strictly forbidding directory traversal sequences (`../`, absolute paths, symlinks, NTFS alternate data streams).
- **Rationale:** Protects against "Zip Slip" vulnerabilities where a malicious wallpaper package overwrites system files or auto-start programs.
- **Acceptance Criteria:**
  1. Archive extraction algorithm verifies that every target canonical path resides strictly within the designated destination directory.
  2. Attempts to unpack files containing `..` or leading drive letters trigger immediate extraction abort and error alert.
- **Verification Method:** Security test attempting to extract a maliciously crafted zip file containing `../../evil.exe`; assert extraction fails with path traversal error.
- **Dependencies:** Secure archive unpacking library.
- **Risks & Constraints:** None.

#### KW-NF-015: Dependency & Supply Chain Security Audit
- **Title:** Automated Supply Chain Vulnerability Auditing
- **Classification:** TECHNICAL-DECISION
- **Priority:** P1
- **Status:** PLANNED
- **Source:** Technical Decisions, Section 7.3
- **Statement:** All third-party Rust crates and frontend dependencies must undergo continuous automated security audits (`cargo audit`, `npm audit`) with zero high-severity or critical vulnerabilities allowed in production releases.
- **Rationale:** Ensures dependencies are free from known exploits and CVEs.
- **Acceptance Criteria:**
  1. CI pipeline runs `cargo audit` and `cargo deny` on every build.
  2. Builds fail if any unpatched high-severity CVE is detected in the dependency graph.
- **Verification Method:** Automated CI security scan step execution.
- **Dependencies:** `cargo-audit`, `cargo-deny`.
- **Risks & Constraints:** Requires timely updates when upstream crate vulnerabilities are published.

#### KW-NF-016: Safe Web Sandbox & Local Storage Segregation
- **Title:** Web Runtime Sandbox Isolation
- **Classification:** USER-REQUIRED
- **Priority:** P0
- **Status:** PLANNED
- **Source:** User Project Brief, Section 7.3 & KW-F-018
- **Statement:** The web wallpaper runtime must enforce complete isolation from host credentials, local cookies, and stored browser passwords, running in an isolated ephemeral or dedicated user data directory.
- **Rationale:** Prevents web wallpapers from accessing user web sessions or stored credentials in Microsoft Edge.
- **Acceptance Criteria:**
  1. WebView2 user data folder points to an isolated directory `%LOCALAPPDATA%\KinetiWall\web_cache`.
  2. Web runtime cannot read standard Edge browser cookies, history, or saved passwords.
- **Verification Method:** Inspection of WebView2 initialization parameters and file access permissions.
- **Dependencies:** KW-F-017, KW-F-018.
- **Risks & Constraints:** None.

---

### 3.4 Privacy & User Sovereignty

#### KW-NF-017: Zero Telemetry, Analytics, or Behavioral Tracking
- **Title:** Absolute Telemetry Prohibition
- **Classification:** USER-REQUIRED
- **Priority:** P0
- **Status:** PLANNED
- **Source:** User Project Brief, Section 7.4 & KW-NG-002
- **Statement:** KinetiWall must contain zero telemetry SDKs, analytics tracking code, user behavior logging, or network reporting mechanisms. No information regarding user wallpapers, runtime hours, hardware specifications, or IP addresses may be collected or transmitted.
- **Rationale:** User customization utilities have no legitimate justification for tracking desktop habits; KinetiWall is strictly committed to total user privacy.
- **Acceptance Criteria:**
  1. Codebase audit confirms complete absence of telemetry libraries (e.g., Google Analytics, Sentry, Mixpanel).
  2. Network packet capture (Wireshark) over 24 hours of active operation reveals zero outbound network connections initiated by the core engine.
- **Verification Method:** Static binary string/dependency inspection and dynamic Wireshark packet capture analysis.
- **Dependencies:** KW-NG-002.
- **Risks & Constraints:** Crash debugging must rely on local opt-in log export rather than automated remote crash dump uploading.

#### KW-NF-018: Desktop Privacy & Anti-Screen-Scraping Guarantee
- **Title:** No Desktop Capture or Screenshot Collection
- **Classification:** USER-REQUIRED
- **Priority:** P0
- **Status:** PLANNED
- **Source:** User Project Brief, Section 7.4
- **Statement:** The application must never capture screenshots, scrape desktop pixels, inspect open application window contents, or read user document paths on the desktop.
- **Rationale:** Desktop customization tools must not act as covert surveillance tools or spyware.
- **Acceptance Criteria:**
  1. Window tracking routines query only window boundaries, process names, and window styles; they never invoke `PrintWindow`, `BitBlt` on desktop HDCs, or Windows Graphics Capture APIs.
- **Verification Method:** Source code inspection verifying no screen capture APIs exist in the codebase.
- **Dependencies:** KW-F-025.
- **Risks & Constraints:** None.

#### KW-NF-019: Zero Required Online Accounts or Registration
- **Title:** Account-Free Sovereign Operation
- **Classification:** USER-REQUIRED
- **Priority:** P0
- **Status:** PLANNED
- **Source:** User Project Brief, Section 7.4 & KW-NG-003
- **Statement:** The user must never be prompted to create an account, enter an email address, authenticate via OAuth, or sign in to use any core feature of KinetiWall.
- **Rationale:** Ensures complete user freedom and friction-free setup without vendor lock-in.
- **Acceptance Criteria:**
  1. Installation, startup, wallpaper playback, and settings customization proceed with zero account prompts or registration dialogs.
- **Verification Method:** Manual user flow audit through full installation and feature lifecycle.
- **Dependencies:** KW-NG-003.
- **Risks & Constraints:** None.

---

### 3.5 Usability, Ergonomics & Accessibility

#### KW-NF-020: Beginner-Friendly vs Advanced Customization Hierarchy
- **Title:** Progressive Disclosure User Interface Design
- **Classification:** USER-REQUIRED
- **Priority:** P1
- **Status:** PLANNED
- **Source:** User Project Brief, Section 7.5
- **Statement:** The user interface must employ progressive disclosure: beginner users can select, play, and mute wallpapers in one click, while advanced settings (multi-monitor spanning, shader parameters, power policies, debug logs) are logically organized in dedicated sub-views.
- **Rationale:** Prevents overwhelming casual users while providing deep customization for enthusiasts.
- **Acceptance Criteria:**
  1. First launch presents a clean, immediate visual grid of wallpapers with an obvious "Apply" interaction.
  2. Advanced configuration tabs are accessible but do not obstruct primary selection workflows.
- **Verification Method:** Usability walkthrough verifying key task completion within 3 clicks from cold launch.
- **Dependencies:** KW-F-007.
- **Risks & Constraints:** None.

#### KW-NF-021: Clear User-Facing Error Notifications & Actionable Feedback
- **Title:** Actionable Diagnostic Error Messaging
- **Classification:** USER-REQUIRED
- **Priority:** P1
- **Status:** PLANNED
- **Source:** User Project Brief, Section 7.5
- **Statement:** When an error occurs (unsupported video format, failed desktop attachment, corrupted package), KinetiWall must display a clear, plain-language notification explaining the problem and offering an actionable resolution (e.g., "Install HEVC codec", "Reset wallpaper"). Raw crash stack traces must be relegated to diagnostic log files.
- **Rationale:** Cryptic error codes frustrate users and generate avoidable support tickets.
- **Acceptance Criteria:**
  1. Errors surfaced in the UI contain a title, concise explanation, and primary action button.
  2. Technical error details and Win32 HRESULTs are saved to `%LOCALAPPDATA%\KinetiWall\logs\kinetiwall.log`.
- **Verification Method:** Fault injection test validating UI error dialog presentation and log file contents.
- **Dependencies:** KW-F-007, KW-F-041.
- **Risks & Constraints:** None.

#### KW-NF-022: Keyboard Navigation & Screen Reader Accessibility
- **Title:** Accessible Keyboard Navigation
- **Classification:** TECHNICAL-DECISION
- **Priority:** P2
- **Status:** PLANNED
- **Source:** Technical Decisions, Section 7.5
- **Statement:** The gallery UI must support full keyboard navigation (Tab, Arrow keys, Enter, Esc), visible focus rings, and standard ARIA attributes for screen reader compatibility.
- **Rationale:** Ensures accessibility for users relying on keyboard navigation or assistive technology.
- **Acceptance Criteria:**
  1. All gallery items, buttons, and sliders can be navigated and activated using keyboard controls exclusively.
  2. Active focus indicator is clearly visible on focused elements.
- **Verification Method:** Automated accessibility audit and manual keyboard-only navigation test.
- **Dependencies:** KW-F-007.
- **Risks & Constraints:** None.

---

### 3.6 Operating System Compatibility & Deployment

#### KW-NF-023: Target Windows Operating System Baseline
- **Title:** Supported Windows OS Matrix
- **Classification:** USER-REQUIRED
- **Priority:** P0
- **Status:** PLANNED
- **Source:** User Project Brief, Section 7.6 & UDR-001
- **Statement:** KinetiWall must officially support:
  - **Windows 10 (64-bit):** Version 2004 (Build 19041) and newer (20H2, 21H1, 21H2, 22H2).
  - **Windows 11 (64-bit):** All versions (Build 22000, 22H2/22621, 23H2/22631, 24H2/26100+).
  - Architecture: **x86_64** (AMD64) as primary target; ARM64 planned for future evaluation.
- **Rationale:** Windows 10 Build 19041 is the minimum version supporting modern Win32 APIs, WebView2 integration, and Per-Monitor V2 DPI features.
- **Acceptance Criteria:**
  1. Application compiles, installs, and runs on verified Windows 10 22H2 and Windows 11 23H2/24H2 systems.
  2. Windows versions prior to Windows 10 Build 19041 display a clean incompatibility warning upon installer launch and exit safely.
- **Verification Method:** Execution testing across automated virtual machines running Windows 10 (19041), Windows 10 (19045), and Windows 11 (22631, 26100).
- **Dependencies:** Windows Win32 API.
- **Risks & Constraints:** Windows 11 24H2 introduces shell changes to desktop icons; WorkerW attachment must be thoroughly verified on Windows 11 builds (see UDR-002).

#### KW-NF-024: Clean Installation & Uninstallation Footprint
- **Title:** Zero-Artifact Clean Uninstallation
- **Classification:** USER-REQUIRED
- **Priority:** P0
- **Status:** PLANNED
- **Source:** User Project Brief, Section 7.8
- **Statement:** The uninstaller must cleanly and completely remove all installed executables, registered shell shortcuts, startup registry keys, and offer the user an option to remove or keep user wallpapers and configuration data.
- **Rationale:** Avoids polluting user systems with leftover registry keys or orphaned background daemons.
- **Acceptance Criteria:**
  1. Running the uninstaller terminates all active KinetiWall processes.
  2. Removes all application binaries from the install directory.
  3. Deletes `HKCU\Software\Microsoft\Windows\CurrentVersion\Run\KinetiWall`.
  4. Restores standard Windows desktop wallpaper.
- **Verification Method:** Clean install and uninstall cycle followed by file system and registry scanning for leftover artifacts.
- **Dependencies:** Installer framework.
- **Risks & Constraints:** None.

#### KW-NF-025: Self-Contained Native Build & Toolchain Reproducibility
- **Title:** Reproducible Native Rust Toolchain Builds
- **Classification:** TECHNICAL-DECISION
- **Priority:** P1
- **Status:** PLANNED
- **Source:** Technical Decisions, Section 7.7
- **Statement:** The core controller and player components must build reproducibly using the stable Rust toolchain (`stable-x86_64-pc-windows-msvc`) with locked dependencies (`Cargo.lock`).
- **Rationale:** Ensures long-term maintainability, deterministic builds, and auditability.
- **Acceptance Criteria:**
  1. `cargo build --release` produces all core binaries without external unmanaged script dependencies.
  2. Build process finishes in under 3 minutes on standard multi-core development workstations.
- **Verification Method:** Clean environment build test executing `cargo clean` followed by `cargo build --release`.
- **Dependencies:** Rust toolchain 1.80+.
- **Risks & Constraints:** Third-party C/C++ libraries (if building libmpv from source) require MSVC/Clang build tools.

---

### 3.7 Maintainability & Code Architecture

#### KW-NF-026: Modular Rust Multi-Crate Architecture
- **Title:** Modular Cargo Workspace Structure
- **Classification:** TECHNICAL-DECISION
- **Priority:** P0
- **Status:** PLANNED
- **Source:** Technical Decisions, Section 7.7
- **Statement:** The KinetiWall codebase must be organized as a modular Cargo workspace separating distinct architectural concerns into isolated crates:
  - `kinetiwall-core`: Main daemon, tray icon, supervisor, state engine.
  - `kinetiwall-player-video`: Dedicated video playback engine.
  - `kinetiwall-player-web`: Dedicated WebView2 interactive player.
  - `kinetiwall-ipc`: Shared IPC protocol contracts, pipe listeners, client wrappers.
  - `kinetiwall-common`: Common types, logging, Win32 helpers, error definitions.
- **Rationale:** Prevents monolithic coupling, speeds up incremental compilation, and enforces strict boundary discipline.
- **Acceptance Criteria:**
  1. Root `Cargo.toml` declares workspace members.
  2. No circular dependencies exist between workspace crates.
  3. Crate boundaries match process execution boundaries.
- **Verification Method:** `cargo tree` and `cargo check --workspace` validation.
- **Dependencies:** Cargo Workspace.
- **Risks & Constraints:** None.

#### KW-NF-027: Automated Test Coverage Baseline
- **Title:** Unit and Integration Test Automation
- **Classification:** TECHNICAL-DECISION
- **Priority:** P1
- **Status:** PLANNED
- **Source:** Technical Decisions, Section 7.7
- **Statement:** Core logic (IPC serialization, configuration schema validation, monitor layout geometry, power state logic) must achieve a minimum of **75% automated unit test coverage**.
- **Rationale:** Guarantees regression prevention across subsequent development steps.
- **Acceptance Criteria:**
  1. `cargo test --workspace` executes all unit and integration tests cleanly.
  2. Code coverage reports (`cargo-tarpaulin` or `llvm-cov`) report ≥ 75% coverage across core modules.
- **Verification Method:** Execution of automated test harness in CI.
- **Dependencies:** Rust standard test harness.
- **Risks & Constraints:** Win32 GUI window interactions require mock window handles or specialized test runners.

---

## 4. Requirements Summary Metrics

| Requirement Classification | Count | P0 (Critical) | P1 (High) | P2 (Medium) | P3 (Low) | Unassigned |
| :--- | :---: | :---: | :---: | :---: | :---: | :---: |
| **USER-REQUIRED** | 47 | 33 | 14 | 0 | 0 | 0 |
| **TECHNICAL-DECISION** | 22 | 12 | 9 | 1 | 0 | 0 |
| **RESEARCH-RECOMMENDATION** | 4 | 0 | 1 | 2 | 1 | 0 |
| **NON-GOAL** | 6 | 6 | 0 | 0 | 0 | 0 |
| **TOTAL SPECIFIED REQUIREMENTS** | **79** | **51** | **24** | **3** | **1** | **0** |
