# KinetiWall — Risk & Validation Register

**Document Identifier:** KW-RES-004  
**Document Type:** Technical Risk Assessment & Proof-of-Concept Protocol (Deliverable D)  
**Current Phase:** Phase 1 — Project Research, Requirements & Feasibility  
**Step:** Step 2 of 90 — Technical Research & Source Verification  
**Status:** Approved Risk Register  
**Date of Research:** 2026-11-08  

---

## 1. Risk Evaluation & Proof-of-Concept (PoC) Governance

This register tracks high-consequence technical uncertainties, potential platform failure modes, and required empirical proofs of concept (PoCs) required before full production implementation begins.

### Risk Severity Levels:
- **CRITICAL:** High probability of blocking core functionality; could require architectural redesign if unmitigated.
- **HIGH:** Significant platform, licensing, or performance hazard; requires dedicated prototype validation.
- **MEDIUM:** Operational, usability, or compatibility risk; manageable through established engineering design patterns.
- **LOW:** Minor performance or edge-case bug; addressable through standard defensive programming.

---

## 2. Technical Risk & Proof-of-Concept Matrix

### TR-001: Windows 11 24H2 Desktop Composition & WorkerW Breakage
- **Severity:** **CRITICAL**
- **Impact Area:** Desktop Window Attachment (`KW-F-001`, `UDR-002`)
- **Uncertainty / Failure Mode:** In Windows 11 Version 24H2, `explorer.exe` composition changes cause the traditional `WorkerW` spawned by message `0x052C` to detach or vanish if desktop icons are hidden, leading to infinite refresh flashing or grey desktops.
- **Mitigation Strategy:** Implement dual-mode desktop attachment:
  1. Inspect whether desktop icons are enabled via `SHELLDLL_DefView` presence.
  2. If icons are hidden or standard WorkerW parenting fails, attach directly beneath `Progman` using `SetWindowPos(..., SWP_NOACTIVATE)`.
  3. Enforce a 500ms debounce timer on Explorer attachment recovery.
- **Required Proof-of-Concept (PoC):** Phase 1, Step 3 (Windows Desktop Architecture Spike).
- **PoC Exit Criteria:** A standalone Rust Win32 prototype successfully parents a window beneath desktop icons on Windows 11 24H2 with icons both enabled and disabled; icon dragging and clicking remains 100% operational.

---

### TR-002: Video Loop Boundary Frame Stutter & Black Flashes
- **Severity:** **HIGH**
- **Impact Area:** Video Playback & Media Engine (`KW-F-013`, `UDR-004`)
- **Uncertainty / Failure Mode:** Standard media player looping flushes Direct3D swapchains at EOF, resulting in a 1-frame black flash or micro-stutter (>30 ms delay).
- **Mitigation Strategy:** Standardize on `libmpv` with `vo=gpu-next` and `hwdec=d3d11va`, setting `loop-file=inf` to ensure the Direct3D 11 texture presentation context remains alive across file loops.
- **Required Proof-of-Concept (PoC):** Phase 1, Step 5 (Media Engine & Gapless Loop Spike).
- **PoC Exit Criteria:** High-speed frame-time profiling over 100 consecutive loops of a 1080p60 H.264 video demonstrates zero frames exceeding 33.3 ms (<1 frame jitter at 60Hz) and zero cleared/black frames.

---

### TR-003: Media Library Redistribution & Licensing Compliance
- **Severity:** **HIGH**
- **Impact Area:** Media Licensing & Redistribution (`UDR-005`)
- **Uncertainty / Failure Mode:** Distributing pre-compiled community builds of `libmpv` or `FFmpeg` that link GPL-licensed filters (e.g. `libx264`) legally infects the application codebase under copyleft GPL v3.0 terms.
- **Mitigation Strategy:** Build or obtain verified `libmpv-2.dll` binaries compiled strictly with `--enable-lgpl --disable-gpl --disable-nonfree`. Dynamically link via C-FFI runtime calls (`LoadLibraryExW`) without static library linking. Include LGPL 2.1 notices and source build instructions in the installer.
- **Required Proof-of-Concept (PoC):** Phase 1, Step 5 (Licensing & Build Audit).
- **PoC Exit Criteria:** Dependency license scan (`cargo deny`) and binary symbol inspection confirm zero GPL-only symbols in `libmpv-2.dll`.

---

### TR-004: WebView2 Virtual Host Mapping & Relative Asset Resolution
- **Severity:** **MEDIUM**
- **Impact Area:** Interactive Web Wallpapers (`KW-F-018`, `UDR-010`)
- **Uncertainty / Failure Mode:** Third-party web wallpapers using absolute paths (e.g. `/assets/shader.frag`) or Web Workers may fail to resolve assets when mapped through `SetVirtualHostNameToFolderMapping`.
- **Mitigation Strategy:** Standardize virtual host mapping to `https://kinetiwall.local/` with `COREWEBVIEW2_HOST_RESOURCE_ACCESS_KIND_DENY_CORS`. Inject a base URL tag (`<base href="https://kinetiwall.local/">`) into the DOM before scripts execute if relative paths are misconfigured.
- **Required Proof-of-Concept (PoC):** Phase 1, Step 4 (Web Player Sandbox Prototype).
- **PoC Exit Criteria:** WebGL2 particle simulation and multi-asset HTML5 wallpaper load all local textures and audio files cleanly without CORS errors or console warnings.

---

### TR-005: WASAPI Loopback Endpoint Invalidation on Device Switch
- **Severity:** **MEDIUM**
- **Impact Area:** Audio Visualization Subsystem (`KW-F-031`, `KW-F-033`)
- **Uncertainty / Failure Mode:** When the user connects or disconnects headphones, the active audio render endpoint is invalidated with `AUDCLNT_E_DEVICE_INVALIDATED`, causing the audio capture thread to crash or permanently hang.
- **Mitigation Strategy:** Implement `IMMNotificationClient::OnDefaultDeviceChanged`. When a device change event fires, cleanly release the old `IAudioClient`, query the new default device, and re-initialize the loopback stream within 500 ms.
- **Required Proof-of-Concept (PoC):** Phase 1, Step 6 (Audio Loopback & FFT Spike).
- **PoC Exit Criteria:** Switching Windows default audio playback device during active audio visualization automatically recovers FFT stream within 500 ms without user intervention or process crash.

---

### TR-006: Job Object Handle Inheritance & Zombie Process Leakage
- **Severity:** **MEDIUM**
- **Impact Area:** Process Supervision & Crash Isolation (`KW-NF-010`, `UDR-003`)
- **Uncertainty / Failure Mode:** If the Job Object handle created by `kinetiwall-core.exe` is inherited by child player processes, closing the handle on daemon exit will not reduce the kernel handle count to zero, defeating `JOB_OBJECT_LIMIT_KILL_ON_JOB_CLOSE`.
- **Mitigation Strategy:** Enforce `bInheritHandle = FALSE` in `SECURITY_ATTRIBUTES` during `CreateJobObjectW`. Verify child process creation explicitly flags `bInheritHandles = FALSE` in `CreateProcessW`.
- **Required Proof-of-Concept (PoC):** Phase 1, Step 3 (Process Lifecycle Spike).
- **PoC Exit Criteria:** Terminating `kinetiwall-core.exe` via `Stop-Process -Force` in PowerShell terminates all child player processes within 100 ms.

---

### TR-007: Mixed-DPI Scaling & Virtual Monitor Coordinate Mapping
- **Severity:** **MEDIUM**
- **Impact Area:** Multi-Monitor Management (`KW-F-020`, `KW-F-024`, `UDR-009`)
- **Uncertainty / Failure Mode:** In setups with mismatched DPI (e.g. 200% 4K laptop + 100% 1080p monitor), Windows DWM virtualizes coordinates, causing player windows to blur or render at incorrect physical offsets.
- **Mitigation Strategy:** Declare `PerMonitorV2` DPI awareness in the application manifest. Intercept `WM_DPICHANGED` and recalculate viewport bounds using `GetDpiForMonitor` and physical pixel dimensions.
- **Required Proof-of-Concept (PoC):** Phase 1, Step 3 (Multi-Monitor Coordinate Spike).
- **PoC Exit Criteria:** Multi-monitor test fixture validates correct viewport bounds and crisp non-blurred presentation across simulated mixed-DPI displays.

---

### TR-008: Named Pipe Buffer Saturation during 60 FPS Audio Streaming
- **Severity:** **LOW**
- **Impact Area:** Inter-Process Communication (`KW-F-008`, `KW-F-032`)
- **Uncertainty / Failure Mode:** Streaming 60 FPS audio FFT magnitude arrays over the same Named Pipe as high-priority control commands could saturate pipe buffers and delay pause/stop commands.
- **Mitigation Strategy:** Decouple IPC channels into two distinct Named Pipes:
  1. `\\.\pipe\kinetiwall-ipc-control` — Low-latency, high-priority command channel.
  2. `\\.\pipe\kinetiwall-ipc-audio` — High-throughput streaming data channel with non-blocking drops if the consumer lags.
- **Required Proof-of-Concept (PoC):** Phase 1, Step 4 (IPC Performance Benchmark).
- **PoC Exit Criteria:** Continuous 60 FPS audio data streaming does not delay transport control commands (Play/Pause) beyond 15 ms.

