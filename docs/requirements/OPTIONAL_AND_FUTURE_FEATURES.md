# KinetiWall — Optional & Future Features Catalog

**Document Identifier:** KW-DOC-005  
**Document Type:** Product Roadmap & Candidate Feature Catalog  
**Current Phase:** Phase 1 — Project Research, Requirements & Feasibility  
**Step:** Step 1 of 90 — Project Requirements Inventory & Traceability  
**Status:** Approved Roadmap Working Catalog  
**Date:** 2026-11-08  

---

## 1. Governance & Scope Separation Policy

To preserve the focus, stability, and delivery timeline of the core KinetiWall v1.0 release baseline, all advanced, exploratory, speculative, or community-inspired features identified in Gemini research, competitor audits, and user discussions are recorded in this dedicated **Optional & Future Features Catalog**.

### Rules of Engagement:
1. **Strict Non-Mandatory Status:** No feature in this catalog is part of the mandatory v1.0 release baseline.
2. **No Silent Upgrades:** A candidate feature cannot be promoted to the active development inventory without explicit user authorization and formal architectural review.
3. **Prerequisite Gating:** Future features must not compromise core product tenets: low memory overhead, battery awareness, rock-solid stability, and zero telemetry.

---

## 2. Catalog of Candidate & Future Features

### OFF-001: Dynamic Windows Accent-Color Synchronization
- **Title:** Adaptive Windows System Accent Color Extraction
- **Classification:** OPTIONAL-ENHANCEMENT
- **Priority:** P2
- **Proposed Release Milestone:** v1.1 / Post-v1
- **Source Document:** Research Document & Competitor Audit (Lively Wallpaper)
- **Concept:** Periodically extract the dominant or vibrant color palette from the currently playing live wallpaper frame and dynamically update the Windows system accent color (`HKCU\Software\Microsoft\Windows\DWM\AccentColor`) and taskbar/start menu tinting.
- **Dependencies:** KW-F-010, Win32 DWM APIs (`DwmSetColorizationParameters` / Windows Registry).
- **Feasibility & Architectural Impact:** Video frame color sampling can consume CPU if performed at high frequency. Must be rate-limited (e.g., once every 5 seconds or upon scene change).
- **Current Approval Status:** Awaiting formal approval; deferred from initial release.

---

### OFF-002: Quick-Switcher Heads-Up Display (HUD)
- **Title:** Keyboard Shortcut Quick-Switcher Overlay
- **Classification:** OPTIONAL-ENHANCEMENT
- **Priority:** P2
- **Proposed Release Milestone:** v1.1
- **Source Document:** Research Document (User Productivity Concepts)
- **Concept:** A lightweight, transparent HUD overlay triggered via a global hotkey (e.g., `Win+Ctrl+W`) allowing users to cycle through favorite wallpapers, toggle audio mute, or pause playback without opening the full gallery window.
- **Dependencies:** Win32 Global Hotkeys (`RegisterHotKey`), layered popup window.
- **Feasibility & Architectural Impact:** Lightweight and straightforward to implement in native Rust; low memory footprint.
- **Current Approval Status:** Recommended for v1.1 milestone.

---

### OFF-003: Time-Based Wallpaper Scheduling & Playlist Automation
- **Title:** Chronological Scheduling & Wallpaper Rotation Playlists
- **Classification:** OPTIONAL-ENHANCEMENT
- **Priority:** P2
- **Proposed Release Milestone:** v1.1
- **Source Document:** Research Document & User Scoping
- **Concept:** Allow users to build wallpaper playlists that rotate on a timer (e.g., every 30 minutes), transition based on time of day (day/night wallpapers matching local solar time), or respond to Windows Dark/Light mode toggles.
- **Dependencies:** KW-F-035, Windows Day/Night theme registry hooks.
- **Feasibility & Architectural Impact:** Core scheduler can run within `kinetiwall-core` using low-frequency timers without performance impact.
- **Current Approval Status:** High-priority candidate for post-v1 update.

---

### OFF-004: In-App Wallpaper Creation & Video Trimming Tool
- **Title:** Integrated Video Trimming and Loop Maker
- **Classification:** OPTIONAL-ENHANCEMENT
- **Priority:** P3
- **Proposed Release Milestone:** v1.2+
- **Source Document:** Competitor Audit (Wallpaper Engine Editor)
- **Concept:** A visual utility inside the gallery UI allowing users to import raw video files, select start/end points, apply crossfade transitions at the loop boundary, and export an optimized seamless `.mp4` wallpaper package.
- **Dependencies:** FFmpeg CLI or libavfilter processing pipelines.
- **Feasibility & Architectural Impact:** High UI and encoding complexity. Bundling video encoders (e.g., x264/x265) bloats the application footprint and triggers licensing complexities.
- **Current Approval Status:** Deferred; users can trim videos externally in third-party tools (Handbrake, LosslessCut) for v1.

---

### OFF-005: Shadertoy GLSL Import & Compatibility Layer
- **Title:** Automated Shadertoy Shader Import & Uniform Translation
- **Classification:** RESEARCH-RECOMMENDATION
- **Priority:** P3
- **Proposed Release Milestone:** v1.2
- **Source Document:** Research Document (Shader Exploration)
- **Concept:** An import filter that accepts Shadertoy GLSL code fragments (`iResolution`, `iTime`, `iMouse`), automatically wraps them into a compatible WebGL2 or native DirectX pixel shader harness, and generates a runnable wallpaper.
- **Dependencies:** WebGL2 shader harness or native Direct3D shader compiler (`d3dcompiler.dll`).
- **Feasibility & Architectural Impact:** WebGL2 wrapper approach is low-risk; native HLSL translation is highly complex.
- **Current Approval Status:** Experimental research recommendation; non-blocking for v1.

---

### OFF-006: Hardware Telemetry Desktop Widgets
- **Title:** Real-Time CPU, GPU, RAM & Network Overlay Widgets
- **Classification:** OPTIONAL-ENHANCEMENT
- **Priority:** P3
- **Proposed Release Milestone:** v1.2+
- **Source Document:** Research Document & Competitor Audit
- **Concept:** Provide customizable desktop telemetry widgets (CPU load graphs, GPU temperature, RAM usage, network speeds) rendered as interactive overlays over any live wallpaper.
- **Dependencies:** Windows Performance Data Helper (PDH) / WMI / LibreHardwareMonitor bindings, KW-F-019.
- **Feasibility & Architectural Impact:** Polling hardware sensors at high frequency causes background CPU wakeups and potential anti-cheat flags. Must be implemented with strict low-frequency polling.
- **Current Approval Status:** Awaiting user demand; deferred from core v1.

---

### OFF-007: Standardized Portable Wallpaper Package Format (`.kwpack`)
- **Title:** Self-Contained `.kwpack` Distribution Archive
- **Classification:** TECHNICAL-DECISION
- **Priority:** P2
- **Proposed Release Milestone:** v1.1
- **Source Document:** Technical Architecture Scoping
- **Concept:** Define a standardized, compressed zip-based package format (`.kwpack` or `.zip`) containing the media asset, thumbnail, property manifest (`kinetiwall.json`), and author metadata for seamless one-click sharing and import.
- **Dependencies:** KW-F-034, KW-F-039, KW-NF-014 (Zip Slip protection).
- **Feasibility & Architectural Impact:** High value, low technical risk. Clean extension of the JSON manifest architecture.
- **Current Approval Status:** Planned for early post-v1 standardization.

---

### OFF-008: Community Wallpaper Distribution & Online Repository
- **Title:** Online Community Browser & One-Click Download
- **Classification:** FUTURE-RELEASE
- **Priority:** P3
- **Proposed Release Milestone:** v2.0+
- **Source Document:** Research Document (Competitor Ecosystems)
- **Concept:** An integrated online catalog allowing creators to publish and users to browse, search, and install community-created wallpapers directly from within the gallery UI.
- **Dependencies:** Hosted cloud backend, CDN infrastructure, user authentication, moderation system, content licensing.
- **Feasibility & Architectural Impact:** Massive backend, legal, bandwidth, and security operational burdens. Directly violates the offline-first, account-free principle if made mandatory.
- **Current Approval Status:** Explicitly deferred to future major releases; strictly optional if ever implemented.

---

### OFF-009: Native Direct3D Compute-Shader Execution
- **Title:** High-Performance Native Compute Shader Engine
- **Classification:** RESEARCH-RECOMMENDATION
- **Priority:** P3
- **Proposed Release Milestone:** v2.0
- **Source Document:** Research Document (Advanced Graphics Concepts)
- **Concept:** A native Direct3D 12 or Vulkan compute-shader player engine capable of running complex particle simulations and fluid dynamics directly on the GPU without a browser runtime.
- **Dependencies:** DirectX 12 / Vulkan runtime, HLSL/SPIR-V compiler toolchain.
- **Feasibility & Architectural Impact:** Exceptional performance, but requires extensive graphics programming overhead compared to WebView2 WebGL.
- **Current Approval Status:** Pure research exploration; deferred.

---

### OFF-010: RGB Peripheral Lighting Synchronization
- **Title:** Ambient RGB Hardware Synchronization (Razer, Corsair, OpenRGB)
- **Classification:** OPTIONAL-ENHANCEMENT
- **Priority:** P3
- **Proposed Release Milestone:** v1.2+
- **Source Document:** Competitor Audit (Wallpaper Engine iCUE/Chroma Integration)
- **Concept:** Synchronize keyboard, mouse, and ambient room lighting colors with the dominant colors of the currently displayed live wallpaper via vendor SDKs or OpenRGB.
- **Dependencies:** Razer Chroma SDK, Corsair iCUE SDK, OpenRGB C-API.
- **Feasibility & Architectural Impact:** Proprietary SDKs are notoriously bloated and prone to crashes. OpenRGB dynamic loading is preferred.
- **Current Approval Status:** Deferred; evaluated for post-v1 plugin architecture.

---

### OFF-011: Cross-Platform Linux & macOS Desktop Ports
- **Title:** Cross-Platform Desktop Engine Ports
- **Classification:** FUTURE-RELEASE
- **Priority:** P3
- **Proposed Release Milestone:** v2.0+
- **Source Document:** Research Document (Ecosystem Portability)
- **Concept:** Port KinetiWall's core architecture to Linux (X11 root window / Wayland `wlr-layer-shell`) and macOS (`NSWindow` desktop levels).
- **Dependencies:** Completely disparate OS windowing, desktop shell, audio capture, and display APIs.
- **Feasibility & Architectural Impact:** Desktop live wallpaper integration is 100% OS-specific. A Windows-first native architecture cannot be ported without rewriting the entire desktop integration layer.
- **Current Approval Status:** Out of scope for current project lifecycle. Windows desktop remains the exclusive platform.

---

### OFF-012: Local AI-Assisted Frame Interpolation & Upscaling (RIFE / Real-ESRGAN)
- **Title:** Optional Local AI Video Upscaling and 60 FPS Interpolation
- **Classification:** OPTIONAL-ENHANCEMENT
- **Priority:** P3
- **Proposed Release Milestone:** Future Experimental Tooling (Separate CLI)
- **Source Document:** Research Document (AI Technology Exploration)
- **Concept:** An optional standalone CLI tool allowing users with powerful dedicated GPUs to upscale low-resolution video loops or interpolate 30 FPS clips to 60 FPS using neural models (e.g. RIFE, Real-ESRGAN via ONNX Runtime / DirectML) prior to importing into KinetiWall.
- **Dependencies:** DirectML / ONNX Runtime, external model weights.
- **Feasibility & Architectural Impact:** Must never be bundled into the core KinetiWall engine or runtime installers. Evaluated strictly as an independent offline preprocessing utility.
- **Current Approval Status:** Explicitly excluded from core runtime; designated as experimental companion utility.

