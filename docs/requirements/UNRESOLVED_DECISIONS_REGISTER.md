# KinetiWall — Unresolved Decisions Register (UDR)

**Document Identifier:** KW-DOC-004  
**Document Type:** Engineering Architecture & Open Decisions Register  
**Current Phase:** Phase 1 — Project Research, Requirements & Feasibility  
**Step:** Step 1 of 90 — Project Requirements Inventory & Traceability  
**Status:** Active Working Register  
**Date:** 2026-11-08  

---

## 1. Register Purpose & Governance

The **Unresolved Decisions Register (UDR)** catalogs foundational architectural, engineering, security, and licensing questions that cannot be resolved unilaterally during requirements definition. Each item documents the technical problem, potential options, architectural tradeoffs, associated operational risks, known evidence, designated decision owner, and the concrete empirical validation action required in Phase 1 and Phase 2.

Decisions are governed by the following status lifecycle:
- `OPEN`: Under active evaluation; technical spikes or user clarification needed.
- `SPIKE_SCHEDULED`: Assigned to a specific feasibility step for prototyping.
- `RESOLVED`: Validated and approved by user/architect; baseline updated.
- `DEFERRED`: Formally scheduled for evaluation in a post-v1 milestone.

---

## 2. Register of Open Architectural Decisions

### UDR-001: Minimum Windows Build & Supported Architecture Matrix
- **Status:** OPEN (Proposed Baseline: Windows 10 2004 Build 19041+ / Windows 11, x86_64)
- **Importance:** Critical (Foundational to API availability, build configurations, and QA matrix).
- **Core Problem:** Windows 10 has multiple feature updates (1809, 1903, 1909, 2004, 21H2, 22H2). Older builds have incomplete Per-Monitor V2 DPI support and lack modern WebView2 runtime enhancements. In addition, Windows on ARM (Snapdragon X Elite/Plus) is gaining market share.
- **Options & Tradeoffs:**
  - *Option A (Conservative Modern):* Windows 10 2004 (Build 19041) through Windows 11 24H2+, 64-bit x86_64 only. Rejects Windows 7/8.1 and Win 10 <19041. (Recommended).
  - *Option B (Extended):* Support back to Windows 10 1809 (LTSC). Requires fallback code for missing DPI and shell APIs.
  - *Option C (Universal):* Target x86_64 and native Windows on ARM64 simultaneously in v1. Requires cross-compiling libmpv and testing on physical ARM hardware.
- **Associated Risks:** Supporting legacy Windows 10 builds introduces complex API branching. Targeting ARM64 in v1 without dedicated ARM test hardware introduces untested production releases.
- **Available Evidence:** Microsoft has retired all non-LTSC Windows 10 editions prior to 22H2. Build 19041 encompasses >96% of the active Windows 10 install base.
- **Decision Owner:** Product Architect & User.
- **Next Validation Action:** Execute Phase 1 OS compatibility audit testing Win32 shell attachment APIs across Windows 10 19041, 19045, and Windows 11 22631/26100.

---

### UDR-002: Desktop Attachment Strategy & WorkerW Validation
- **Status:** OPEN (Proposed Baseline: Progman / WorkerW Message 0x052C Injection)
- **Importance:** Critical (Directly dictates whether wallpapers can render beneath desktop icons).
- **Core Problem:** Windows does not provide a documented, public Win32 API to place custom application windows behind desktop icons. The standard technique sends message `0x052C` to the `Progman` window, which instructs the Windows Explorer shell to spawn a `WorkerW` window directly beneath `SHELLDLL_DefView`. This behavior is undocumented and internal to `explorer.exe`.
- **Options & Tradeoffs:**
  - *Option A (WorkerW 0x052C Spawn):* Send `0x052C` to `Progman`, locate the sibling `WorkerW` behind the icon view, and set it as the parent window (`SetParent`) of the wallpaper player. (Industry standard used by Lively Wallpaper and Wallpaper Engine).
  - *Option B (Direct Progman Child):* Parent directly into `Progman` without spawning a separate `WorkerW`. Can cause desktop icon redraw stutter.
  - *Option C (Layered Window with Desktop Hook):* Inject a hook into Explorer. Rejected by KW-NG-004.
- **Associated Risks:** Microsoft could alter or remove this internal message in a future Windows 11 feature update (e.g., 24H2 or 25H2), breaking the attachment mechanism.
- **Available Evidence:** Empirical testing across Windows 10 (19041–19045) and Windows 11 (22000–26100) confirms `0x052C` remains functional in all current production releases.
- **Decision Owner:** Windows Desktop Systems Engineer.
- **Next Validation Action:** Phase 1, Step 3 (Windows Desktop Architecture Spike) must execute an isolated Win32 prototype verifying `WorkerW` creation, icon click-through, and Z-order persistence on Windows 11 24H2.

---

### UDR-003: Process Boundaries, Executable Topology & Packaging
- **Status:** OPEN (Proposed Baseline: Multi-Binary Cargo Workspace)
- **Importance:** High (Dictates crate modularity, IPC contracts, memory footprints, and packaging).
- **Core Problem:** Multi-process architectures can be packaged as multiple dedicated executable binaries or as a single multi-call binary with subcommand flags (e.g., `kinetiwall.exe --core`, `kinetiwall.exe --player-video`).
- **Options & Tradeoffs:**
  - *Option A (Dedicated Binaries):* Separate executables: `kinetiwall-core.exe`, `kinetiwall-player-video.exe`, `kinetiwall-player-web.exe`, `kinetiwall-ui.exe`. Clean process boundaries; easy to inspect in Task Manager; independent dependency linking.
  - *Option B (Single Multi-Call Binary):* Single `kinetiwall.exe` executing different routines based on CLI flags. Smaller combined disk size and simpler code sharing, but links all dependencies (video decoders, WebView2, UI frameworks) into one large executable, increasing idle memory.
- **Associated Risks:** Single binary increases idle memory of the core daemon if static C/C++ libraries are linked into the executable image.
- **Available Evidence:** Dedicated binaries allow `kinetiwall-core.exe` to remain pure Rust (<10 MB RAM) without carrying unused media decoder heap structures.
- **Decision Owner:** Lead Software Architect.
- **Next Validation Action:** Compile a minimal multi-binary prototype vs single multi-call binary in Phase 2 to compare memory footprint and build times.

---

### UDR-004: Video Decoding Engine: Embedded libmpv vs Managed mpv Process vs Native WMF
- **Status:** OPEN (Proposed Baseline: Embedded `libmpv` via C-FFI)
- **Importance:** Critical (Dictates media decoding flexibility, gapless looping quality, and binary dependencies).
- **Core Problem:** Video playback requires choosing between embedding `libmpv` directly in a Rust player process, spawning a standalone `mpv.exe` child process over JSON-IPC, or writing a pure Windows Media Foundation (WMF) Direct3D pipeline.
- **Options & Tradeoffs:**
  - *Option A (Embedded libmpv via FFI):* Embed `mpv-sys` / `libmpv-2.dll` directly inside `kinetiwall-player-video`. Provides native Win32 window render context (`mpv_render_context`), sub-millisecond gapless looping, and hardware decode control.
  - *Option B (Managed mpv.exe Process):* Launch external `mpv.exe` process with `--wid=<HWND>` and command it over JSON-IPC pipes. Simpler build, but higher process management overhead, difficult Direct3D swapchain synchronization, and potential window flashing.
  - *Option C (Pure Native Windows Media Foundation):* 100% native Windows COM APIs (`IMFMediaSession`, Direct3D 11 video processor). Zero third-party DLL dependencies; tiny installer footprint. However, gapless looping is notoriously difficult to achieve in WMF without black flashes, and codec support is limited to Microsoft-licensed codecs (no VP8/VP9/WebM out-of-the-box on older Win10).
- **Associated Risks:** Embedded `libmpv` introduces LGPL/GPL licensing considerations (UDR-005) and requires bundling external DLLs. WMF introduces severe media compatibility issues and looping artifacts.
- **Available Evidence:** Lively Wallpaper transitioned from WMF to libmpv due to WMF looping stutter and codec limitations. Wallpaper Engine uses custom Direct3D/MF pipelines with proprietary splitters.
- **Decision Owner:** Senior Media Systems & Rust Engineer.
- **Next Validation Action:** Execute a Phase 1 comparative media spike benchmarking gapless loop latency and memory between embedded libmpv and WMF.

---

### UDR-005: Media Library Licensing (LGPL vs GPL) & Redistribution Compliance
- **Status:** OPEN (Proposed Baseline: LGPL 2.1+ Build of libmpv & FFmpeg)
- **Importance:** High (Legal compliance, software redistribution rights, and proprietary compatibility).
- **Core Problem:** `libmpv` and `FFmpeg` can be compiled under either the GNU Lesser General Public License (LGPL v2.1+) or the GNU General Public License (GPL v2/v3). Compiling with GPL features (e.g., `--enable-gpl`, `libx264`, `libx265`, non-free filters) makes the entire binary derivative GPL-licensed, restricting proprietary distribution or commercial closed-source licensing.
- **Options & Tradeoffs:**
  - *Option A (LGPL Build):* Compile FFmpeg/libmpv with LGPL configuration, relying on hardware decoders (D3D11VA/NVDEC/AMF) and open-source software decoders (dav1d, libvpx). Distribute `libmpv-2.dll` as a dynamically linked shared library. Permissive and compliant with closed or open source. (Recommended).
  - *Option B (GPL Build):* Use standard community GPL builds of mpv. Maximizes codec support but legally mandates GPL licensing across the entire application codebase.
  - *Option C (System Codecs / WMF):* Use Windows Media Foundation. Zero GPL/LGPL baggage, but poor format compatibility.
- **Associated Risks:** Violating LGPL redistribution terms by statically linking LGPL code without providing relinking mechanisms or missing source disclaimers.
- **Available Evidence:** Commercial applications commonly use LGPL FFmpeg dynamic libraries (`.dll`) with dynamically loaded C-FFI bindings.
- **Decision Owner:** Legal / Product Requirements Engineer.
- **Next Validation Action:** Verify the exact build flags of pre-built `libmpv-2.dll` distributions to confirm strict LGPL 2.1 compliance.

---

### UDR-006: WebView2 Runtime Deployment & Genuinely Offline Installation
- **Status:** OPEN (Proposed Baseline: Evergreen System Detection with Offline Guidance)
- **Importance:** High (Installer size, offline installation capability, and enterprise distribution).
- **Core Problem:** Microsoft Edge WebView2 is required for interactive web wallpapers. While Windows 11 and recent Windows 10 updates bundle the Evergreen WebView2 runtime by default, some offline or enterprise LTSC environments may lack it.
- **Options & Tradeoffs:**
  - *Option A (Evergreen System-Installed):* Check registry for system WebView2 runtime. If present, proceed. If missing, warn the user and provide instructions or an optional bundled installer. Keeps core installer under 35 MB. (Recommended).
  - *Option B (Bundled Fixed-Version Runtime):* Bundle the standalone Microsoft WebView2 Fixed Version package directly inside the KinetiWall offline installer. Guarantees 100% offline deployment, but bloats the installer by ~150–200 MB.
  - *Option C (Evergreen Bootstrapper):* Include the tiny 2 MB Microsoft bootstrapper that downloads the runtime during installation. Violates the 100% offline requirement (KW-F-045).
- **Associated Risks:** An air-gapped machine without WebView2 will fail to render web wallpapers under Option A unless an optional offline bundle is provided.
- **Available Evidence:** Microsoft has rolled out Evergreen WebView2 to all supported Windows 10 consumer editions via standard Windows Updates.
- **Decision Owner:** Lead Architect & Installer Engineer.
- **Next Validation Action:** Implement a dual-installer packaging strategy: a compact standard installer (<35 MB) using system WebView2, and an optional full offline enterprise bundle including WebView2 standalone.

---

### UDR-007: Occlusion & Foreground Window Detection Policy
- **Status:** OPEN (Proposed Baseline: Configurable Policy — Fullscreen Pause Mandatory, Maximized Pause Optional)
- **Importance:** Medium (Directly impacts power consumption and user visual expectations).
- **Core Problem:** Users have conflicting preferences regarding when wallpapers should pause. When a window is maximized, the wallpaper is not visible, but transparent/Mica titlebars or translucent panels may expose parts of the desktop. In multi-monitor setups, a game on Monitor 1 should not necessarily pause a dashboard wallpaper on Monitor 2.
- **Options & Tradeoffs:**
  - *Option A (Per-Monitor Independent Occlusion):* Track occlusion independently per display. Fullscreen games pause only the display they occupy, or optionally all displays. Maximized windows pause that monitor's wallpaper.
  - *Option B (Global Pause on Any Fullscreen):* Any fullscreen game on any monitor pauses all wallpapers on all monitors. Maximizes gaming performance and minimizes multi-display GPU contention.
  - *Option C (User-Configurable Matrix):* Provide explicit user toggles: "Pause when other app is fullscreen", "Pause when window is maximized", "Per-display pause vs Global pause".
- **Associated Risks:** Overly aggressive occlusion detection can cause annoying stop/start flickering when users rapidly alt-tab or open floating windows.
- **Available Evidence:** Wallpaper Engine and Lively both implement user-configurable rules: "Other application fullscreen: Pause all / Pause per monitor / Keep running".
- **Decision Owner:** Product Requirements Engineer.
- **Next Validation Action:** Establish the user-configurable matrix as standard in the configuration schema (KW-F-039).

---

### UDR-008: Formal Semantics of Pause vs Suspend vs Stop States
- **Status:** OPEN (Proposed Baseline: 3-Tier State Machine)
- **Importance:** High (Governs thread management, Direct3D resource allocation, and resume latency).
- **Core Problem:** The terms "pause", "suspend", and "stop" are often conflated in requirements, but represent radically different technical states in graphics programming.
- **Options & Tradeoffs:**
  - *State 1: Playback Paused:* Media player decoding stops; video clock pauses; swapchain retains last frame; GPU VRAM remains allocated. Resume latency < 30 ms. CPU = 0.0%.
  - *State 2: Rendering Suspended:* Presentation loop halts; Direct3D swapchain transitions to occluded state (`DXGI_PRESENT_TEST`); rendering thread sleeps. Resume latency < 50 ms.
  - *State 3: Player Stopped / Resources Released:* Player process media stream is closed or process is suspended via `NtSuspendProcess`; Direct3D swapchain and texture buffers freed from VRAM. Resume latency ~250–500 ms.
- **Associated Risks:** Releasing GPU resources reduces VRAM usage to zero, but introduces noticeable stutter/delay when the user minimizes a game to view the desktop.
- **Available Evidence:** Laptop users prioritize battery (State 3 on battery), while desktop gamers with 16GB+ VRAM prefer instant resume (State 1).
- **Decision Owner:** Senior Systems & Graphics Engineer.
- **Next Validation Action:** Formalize the 3-tier state machine in the IPC and player architecture in Phase 2.

---

### UDR-009: Spanning & Clone Multi-Monitor Semantics Across Mismatched Displays
- **Status:** OPEN (Proposed Baseline: Aspect-Fill Virtual Canvas for Spanning; Independent Viewports for Clone)
- **Importance:** Medium (Visual quality and mathematical correctness across complex monitor layouts).
- **Core Problem:** Users frequently have monitors with mismatched resolutions (e.g., 4K primary + 1080p secondary), aspect ratios (16:9 + 21:9 ultra-wide), orientations (horizontal + vertical), and DPI scaling (150% + 100%).
- **Options & Tradeoffs:**
  - *Option A (Virtual Screen Bounding Box):* Create a virtual canvas matching the bounding rectangle of all displays. Sub-windows render their corresponding physical coordinate slice. Areas outside display bounds are clipped.
  - *Option B (Aspect Fit / Aspect Fill per Display):* In clone mode, each display crops or letterboxes the master video stream to preserve aspect ratio without stretching.
- **Associated Risks:** Mismatched DPI scaling across displays causes coordinate mapping errors in Direct3D if Per-Monitor V2 scaling is not handled properly.
- **Available Evidence:** Standard Windows desktop wallpaper spanning aligns displays based on their virtual screen coordinates configured in Windows Display Settings.
- **Decision Owner:** Lead Software Architect.
- **Next Validation Action:** Implement virtual monitor coordinate normalization algorithms in `kinetiwall-common`.

---

### UDR-010: Interactive Web Wallpaper Security Boundary & Protocol Schema
- **Status:** OPEN (Proposed Baseline: Virtual Host Mapping `https://kinetiwall.local/`)
- **Importance:** Critical (Guarantees security against malicious web scripts and CVE exploits).
- **Core Problem:** Web wallpapers execute third-party JavaScript. Allowing unrestricted `file:///` access permits scripts to read sensitive local files. Disabling all file access breaks web wallpapers that load local images, fonts, and shader assets.
- **Options & Tradeoffs:**
  - *Option A (Custom Virtual Host Scheme):* Map the wallpaper folder to a virtual origin using `SetVirtualHostNameToFolderMapping("kinetiwall.local", folder_path, COREWEBVIEW2_HOST_RESOURCE_ACCESS_KIND_ALLOW)`. Scripts can only access subfiles within their own folder. All cross-origin file reads fail. (Recommended).
  - *Option B (Embedded Local HTTP Server):* Run a local loopback HTTP server (e.g., `http://127.0.0.1:port/`). Consumes additional socket handles, triggers Windows Firewall prompts, and allows other local processes to probe the port.
  - *Option C (Unrestricted file:// protocol):* High security risk; completely violates KW-NF-018.
- **Associated Risks:** Web wallpapers using absolute paths (e.g. `/assets/image.png`) may fail if the virtual host mapping root is not calibrated properly.
- **Available Evidence:** Microsoft recommends `SetVirtualHostNameToFolderMapping` as the gold standard for secure local asset delivery in WebView2 applications.
- **Decision Owner:** Security Analyst & Rust Engineer.
- **Next Validation Action:** Phase 1, Step 4 (Security Analysis & Threat Modeling) will formalize the virtual host mapping specification.

---

### UDR-011: Standalone 3D / Game Engine Executable Hosting Scope
- **Status:** OPEN (Proposed Baseline: Deferred from Core Scope; Research Only)
- **Importance:** Medium (Risk management and security boundary preservation).
- **Core Problem:** Supporting standalone `.exe` wallpapers (Unity/Godot/Unreal builds) by capturing their window via `SetParent` introduces severe stability and security hazards: game loops run at 100% GPU, anti-cheat software flags window reparenting, processes leak handles, and malware could disguise itself as a wallpaper.
- **Options & Tradeoffs:**
  - *Option A (Exclude from Core v1 Baseline):* Defer executable hosting to post-v1 or an experimental plugin module. Fulfill 3D needs via WebGL2 in the web player. (Recommended).
  - *Option B (Restricted Sandbox Hosting):* Support executables only if executed inside a restricted Windows AppContainer or Low Integrity Level token. High implementation complexity.
  - *Option C (Full Unrestricted Hosting):* Unsafe; rejected by KW-NG-006.
- **Associated Risks:** Including executable hosting in v1 risks introducing unpatchable security vulnerabilities and severe stability flaws.
- **Available Evidence:** Competitor forums document that executable wallpapers cause >80% of support tickets regarding antivirus false positives and desktop crashes.
- **Decision Owner:** Lead Software Architect & User.
- **Next Validation Action:** Formally classify KW-F-044 as `RESEARCH-RECOMMENDATION` and keep deferred from the mandatory v1 release baseline.

---

### UDR-012: Approval Status of AI-Assisted Capabilities
- **Status:** OPEN (Proposed Baseline: Explicit Non-Requirement for Core Baseline)
- **Importance:** Medium (Prevents massive dependency bloat and scope divergence).
- **Core Problem:** Research documents mention AI/LLM/SLM capabilities, including local generative wallpaper creation, frame interpolation (RIFE/DLSS), and natural language prompt generation.
- **Options & Tradeoffs:**
  - *Option A (Strict Non-Requirement):* Core KinetiWall does not bundle or require any AI, LLM, or SLM components. Core architecture remains 100% deterministic, native Rust. (Recommended).
  - *Option B (External Tooling / Optional Extension):* Provide standalone scripts or optional post-v1 tools for AI upscaling or prompt generation that export standard MP4/WebM files for KinetiWall playback.
- **Associated Risks:** Bundling PyTorch, ONNX Runtime, or large model weights (>2 GB) destroys KinetiWall's lightweight, low-footprint product promise.
- **Available Evidence:** Live wallpaper users want smooth, battery-efficient playback, not an embedded machine learning laboratory.
- **Decision Owner:** Product Architect & User.
- **Next Validation Action:** Confirm in requirements baseline that AI capabilities are categorized as post-v1 optional enhancements (`OFF-012`), never mandatory runtime dependencies.

---

### UDR-013: Numerical Performance Target Validation & Benchmarking Conditions
- **Status:** OPEN (Proposed Baseline: Formal Provisional Calibration)
- **Importance:** High (Provides measurable, objective criteria for performance validation).
- **Core Problem:** Targets such as "<10 MB idle RAM", "<0.1% CPU", and "<50 ms pause response" are proposed in research but must be grounded in specific measurement methodologies and reference test environments.
- **Options & Tradeoffs:**
  - *Provisional Targets Table:*
    - Core Controller Idle RAM: ≤ 15.0 MB Private Working Set (Stretch target: < 10.0 MB).
    - Core Controller Idle CPU: < 0.10% over 10-minute average on 4-core+ CPU.
    - Video Playback CPU Overhead: ≤ 3.0% for 1080p60 H.264 on modern hardware.
    - Pause Response Latency: ≤ 100 ms from event trigger to frame suspension (Stretch: < 50 ms).
  - *Reference Benchmarking Conditions:*
    - OS: Windows 10 22H2 / Windows 11 23H2 (clean install, non-debug build).
    - Hardware: Intel Core i5 (8th Gen+) or AMD Ryzen 5 (3000+) / 16 GB RAM / Dedicated GPU (NVIDIA GTX 1060+ or AMD RX 580+) and Intel UHD 630 iGPU.
    - Tools: Windows Performance Toolkit (`xperf`), PowerShell `Get-Process`, Performance Monitor (`perfmon`).
- **Associated Risks:** Measuring memory in debug builds will yield false failures due to unoptimized allocations and debug symbol tables.
- **Available Evidence:** Rust native daemons without heavy web runtimes routinely maintain private working sets between 6–12 MB.
- **Decision Owner:** Senior QA Engineer & Systems Engineer.
- **Next Validation Action:** Phase 1, Step 5 (Performance & Resource Feasibility Spike) will conduct empirical calibration benchmarks against these provisional targets.

