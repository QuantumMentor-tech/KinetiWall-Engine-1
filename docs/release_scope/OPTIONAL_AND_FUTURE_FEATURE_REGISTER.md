# KinetiWall — Optional and Future Feature Register
**Document ID:** `KW-SCP-005`  
**Deliverable:** Deliverable E — Optional and Future Feature Register  
**Phase:** 1 — Project Research, Requirements & Feasibility  
**Step:** 5 of 90 — Release Scope, Non-Goals & Feature Prioritization  
**Status:** COMPLETE (Auditable Candidate Catalog)  
**Authority:** Technical Research (Step 2) & Competitor Analysis (Step 3)  
**Date:** 2026-11-10  

---

## 1. Purpose & Horizon Strategy

This register documents all candidate features, enhancements, and roadmap opportunities identified during [Step 1 Requirements Inventory](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/requirements/OPTIONAL_AND_FUTURE_FEATURES.md), [Step 2 Technical Research](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/research/README.md), and [Step 3 Competitor Analysis](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/competitive_analysis/README.md).

**Governance Rule:** No feature in this register is part of the mandatory Version 1.0 release commitment. Inclusion in a future release requires formal evaluation against technical feasibility, resource overhead, security risks, and explicit user approval.

```
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                        POST-V1 FEATURE CANDIDATE HORIZONS                              │
├───────────────────────────────┬───────────────────────────────┬────────────────────────┤
│          HORIZON 1            │          HORIZON 2            │       HORIZON 3        │
│   V1.1 – V1.3 MINOR UPDATES   │      V2.0 MAJOR RELEASE       │    RESEARCH / PROTOTYPE │
│   (Quality of Life / Usability)│  (Advanced Authoring & Bridging│  (AI & Niche Hardware) │
├───────────────────────────────┼───────────────────────────────┼────────────────────────┤
│ • Dynamic OS Accent Color     │ • Visual Scene Graph Editor   │ • Local AI Generation  │
│ • Quick-Switcher HUD Overlay  │ • Standalone Game Reparenting │ • AI Video Frame Interp│
│ • Time-of-Day Playlists       │ • Community Workshop Cloud    │ • Linux / Wayland Port │
│ • In-App Video Trimming       │ • Direct Shadertoy Import     │ • Cross-Device LAN Sync│
│ • Hardware Telemetry OSD      │ • Native DX12 Compute Engine  │ • Razer / iCUE RGB Sync│
└───────────────────────────────┴───────────────────────────────┴────────────────────────┘
```

---

## 2. Exhaustive Feature Candidate Registry

The following detailed profiles evaluate all 15 candidate enhancements:

### 2.1 Quality-of-Life Enhancements (Target: V1.x Horizon)

#### `OFF-001`: Dynamic Windows Accent-Color Harmonizer
- **Description:** Periodically extract the dominant 3–5 color palette from the currently playing video/web wallpaper and automatically update the Windows 11 system accent color and window border tinting.
- **Source:** Competitor Analysis (`OPP-A04`), Step 1 Baseline (`OFF-001`).
- **Classification:** `OPTIONAL-ENHANCEMENT` (Priority: `P2`)
- **Intended Users:** Desktop aesthetics enthusiasts seeking visual unity between desktop wallpaper and Windows shell chrome.
- **Expected Value:** Creates a dynamic, cohesive desktop experience similar to macOS dynamic wallpapers.
- **Technical Feasibility:** `HIGH` — Can be achieved via Windows DWM API `DwmSetColorizationColor` and registry keys `HKCU\Software\Microsoft\Windows\DWM`.
- **Security Risks:** Negligible; operates entirely in current user session without elevation.
- **Resource Implications:** Low; color extraction runs once on static images or sampled once every 3 seconds on video (quantized k-means color algorithm takes <5ms CPU).
- **Dependencies:** `KW-F-009` (Video Player), `KW-F-037` (Config Engine).
- **Maintenance Burden:** Low; stable Win32 DWM API.
- **Candidate Release:** `V1.1 CANDIDATE`
- **Acceptance Criteria if Approved:** Wallpaper color changes smoothly transition Windows accent colors; zero UI stutter during transitions.
- **Approval Status:** `AWAITING APPROVAL`

#### `OFF-002`: Quick-Switcher Keyboard HUD Overlay
- **Description:** A lightweight, translucent on-screen overlay triggered by a global keyboard shortcut (e.g. `Win+Alt+W`) allowing instant cycling through favorite wallpapers without opening the main Gallery window.
- **Source:** Competitor Analysis (`OPP-A02`), Step 1 Baseline (`OFF-002`).
- **Classification:** `OPTIONAL-ENHANCEMENT` (Priority: `P2`)
- **Intended Users:** Power users and multi-taskers who switch wallpapers frequently during work/gaming sessions.
- **Expected Value:** Instant wallpaper switching with zero mouse clicks and 0 MB resident RAM impact.
- **Technical Feasibility:** `HIGH` — Global hotkey registered via `RegisterHotKey`, overlay rendered via lightweight GDI/Direct2D borderless window.
- **Security Risks:** Low; hotkey registration must ensure zero conflict with Windows shell shortcuts.
- **Resource Implications:** 0 MB when inactive; overlay window lives for 3 seconds then destroys itself.
- **Dependencies:** `KW-F-005` (Core Daemon), `KW-F-008` (Tray).
- **Maintenance Burden:** Minimal.
- **Candidate Release:** `V1.1 CANDIDATE`
- **Acceptance Criteria if Approved:** Pressing hotkey displays translucent carousel; arrow keys select wallpaper; switching occurs in <200ms.
- **Approval Status:** `AWAITING APPROVAL`

#### `OFF-003`: Circadian Time-of-Day Playlists & Scheduling
- **Description:** Automated wallpaper transitions based on user-defined time rules, local sunrise/sunset calculations, or system power state (e.g., light themes during daytime, dark cybernetic themes after 8:00 PM).
- **Source:** Competitor Analysis (`OPP-A03`), Step 1 Baseline (`OFF-003`).
- **Classification:** `OPTIONAL-ENHANCEMENT` (Priority: `P2`)
- **Intended Users:** Office workers, night-shift coders, and users with circadian lighting setups.
- **Expected Value:** Eliminates manual switching; reduces eye strain during evening hours.
- **Technical Feasibility:** `HIGH` — Background timer in `kinetiwall-core.exe` checks active playlist schedule once per minute.
- **Security Risks:** None.
- **Resource Implications:** Zero; negligible single timer tick every 60 seconds.
- **Dependencies:** `KW-F-005` (Core Daemon), `KW-F-037` (Config Engine).
- **Maintenance Burden:** Low.
- **Candidate Release:** `V1.2 CANDIDATE`
- **Acceptance Criteria if Approved:** Schedule transitions smoothly execute at configured time boundaries with optional cross-fade.
- **Approval Status:** `AWAITING APPROVAL`

#### `OFF-004`: Built-In Video Trimming & Clip Creation Tool
- **Description:** In-gallery trimming interface allowing users to import a full-length video, drag start/end timeline scrubbers, mute audio, and export a perfectly looped 10–30 second MP4 wallpaper.
- **Source:** Competitor Analysis (`OPP-A05`), Step 1 Baseline (`OFF-004`).
- **Classification:** `OPTIONAL-ENHANCEMENT` (Priority: `P2`)
- **Intended Users:** Casual users importing long movie scenes or YouTube downloads without third-party video editors.
- **Expected Value:** Reduces user friction during wallpaper creation; prevents disk bloat from 4 GB video files.
- **Technical Feasibility:** `MEDIUM` — Can leverage bundled `libmpv` or invoke FFmpeg CLI with stream copying (`-c copy`) for instant zero-reencode trimming.
- **Security Risks:** Low; operates strictly on local user-selected video files.
- **Resource Implications:** Ephemeral; CPU/disk I/O active only during trimming execution in Gallery UI.
- **Dependencies:** `KW-NF-002` (Tauri UI), `KW-F-009` (libmpv).
- **Maintenance Burden:** Medium; requires robust UI timeline component in web frontend.
- **Candidate Release:** `V1.2 CANDIDATE`
- **Acceptance Criteria if Approved:** Trimming a 5-minute video to 15 seconds takes <2 seconds via stream copy and creates a valid looped wallpaper.
- **Approval Status:** `AWAITING APPROVAL`

#### `OFF-006`: Hardware Telemetry & Performance OSD Overlay
- **Description:** Real-time system monitoring overlay displaying CPU load, GPU temperature, VRAM usage, and network bandwidth directly integrated onto desktop wallpapers.
- **Source:** Competitor Analysis (`OPP-A01`), Step 1 Baseline (`OFF-006`).
- **Classification:** `OPTIONAL-ENHANCEMENT` (Priority: `P2`)
- **Intended Users:** PC gamers, overclockers, and hardware enthusiasts.
- **Expected Value:** Replaces heavy third-party monitoring widgets (Rainmeter) with hardware-accelerated desktop integration.
- **Technical Feasibility:** `MEDIUM` — Pure-Rust `sysinfo` or Windows PDH (Performance Data Helper) APIs; stats pushed to WebGL canvas.
- **Security Risks:** Requires reading hardware performance counters; zero elevated privilege required.
- **Resource Implications:** Polling hardware counters once every 1,000ms consumes <0.1% CPU.
- **Dependencies:** `KW-F-014` (Web Player), `KW-F-015` (Property Bridge).
- **Maintenance Burden:** Medium; Windows PDH APIs can be quirky across different GPU vendor drivers.
- **Candidate Release:** `V1.3 CANDIDATE`
- **Acceptance Criteria if Approved:** OSD values match Windows Task Manager within 2% margin; CPU impact remains <0.1%.
- **Approval Status:** `AWAITING APPROVAL`

#### `OFF-007`: Application-Specific Pause / Mute Rules Engine
- **Description:** Advanced rules matrix in Gallery UI allowing users to configure custom engine behaviors per executable (e.g. "Mute wallpaper when Chrome plays audio, Pause wallpaper when Premiere Pro is focused, Keep playing when Discord is active").
- **Source:** Competitor Analysis (`OPP-B05`), Step 1 Baseline (`OFF-007`).
- **Classification:** `OPTIONAL-ENHANCEMENT` (Priority: `P2`)
- **Intended Users:** Content creators, streamers, and multi-monitor power users.
- **Expected Value:** Granular control over audio and performance based on exact user workflow.
- **Technical Feasibility:** `HIGH` — Win32 process enumeration upon foreground hook matches active `.exe` against rule table.
- **Security Risks:** None.
- **Resource Implications:** Negligible table lookup (<0.01ms) upon foreground window change.
- **Dependencies:** `KW-F-026` (Occlusion Engine), `KW-F-037` (Config Engine).
- **Maintenance Burden:** Low.
- **Candidate Release:** `V1.3 CANDIDATE`
- **Acceptance Criteria if Approved:** Custom rules execute reliably within 50ms of designated application launch/focus.
- **Approval Status:** `AWAITING APPROVAL`

---

### 2.2 Advanced Platform Expansion (Target: V2.0+ Horizon)

#### `OFF-005`: Direct Shadertoy URL Import & GLSL Transpiler
- **Description:** Paste a Shadertoy URL into KinetiWall to automatically parse, transpile, and bundle the GLSL fragment shader into an offline WebGL wallpaper package with uniform mapping.
- **Source:** Competitor Analysis (`OPP-B02`), Step 1 Baseline (`OFF-005`).
- **Classification:** `FUTURE-OPPORTUNITY` (Priority: `P3`)
- **Intended Users:** Shader enthusiasts, demosceners, and generative art lovers.
- **Expected Value:** Unlocks tens of thousands of free community shaders without manual web packaging.
- **Technical Feasibility:** `MEDIUM` — Requires regex/AST translation for Shadertoy uniform semantics (`iTime`, `iResolution`, `iMouse`, multi-pass buffers).
- **Security Risks:** Untrusted GLSL execution; mitigated by running inside sandboxed WebGL context in WebView2.
- **Resource Implications:** High GPU load if user imports unoptimized raymarching shaders; requires GPU timeout protection.
- **Dependencies:** `KW-F-014` (WebView2 Player), `KW-F-016` (Sandboxing).
- **Maintenance Burden:** High; Shadertoy API and multi-buffer shaders evolve over time.
- **Candidate Release:** `V2.0+ CANDIDATE`
- **Acceptance Criteria if Approved:** Standard single-pass Shadertoy URLs convert and render at 60 FPS in <3 seconds.
- **Approval Status:** `AWAITING APPROVAL`

#### `OFF-008`: Community Wallpaper Workshop & Cloud Sharing
- **Description:** Decentralized or curated community repository allowing creators to publish, browse, review, and download verified live wallpaper packages with one-click installation.
- **Source:** Competitor Analysis (`GAP-01`, `OPP-C01`), Step 1 Baseline (`OFF-008`).
- **Classification:** `FUTURE-OPPORTUNITY` (Priority: `P3`)
- **Intended Users:** General consumers seeking a vast ecosystem of community-generated live wallpapers.
- **Expected Value:** Bridges the largest competitive gap with Wallpaper Engine's Steam Workshop.
- **Technical Feasibility:** `MEDIUM` — Backend API for package hosting (Cloudflare R2 / GitHub Releases) + frontend browsing UI in Tauri.
- **Security Risks:** `HIGH` — User-submitted packages could contain malicious web scripts or copyright-infringing media; requires automated static analysis and sandboxed execution.
- **Resource Implications:** Moderate; network bandwidth during download; zero background resource impact when closed.
- **Dependencies:** `KW-F-032` (Library), `KW-NF-016` (Security Sandbox), `V1-EXCL-002`.
- **Maintenance Burden:** High; ongoing server infrastructure costs, content moderation, DMCA takedown handling.
- **Candidate Release:** `V2.0+ CANDIDATE`
- **Acceptance Criteria if Approved:** One-click download unpacks and registers wallpaper cleanly; zero malware escape from sandbox.
- **Approval Status:** `AWAITING APPROVAL`

#### `OFF-009`: In-App Visual Scene-Graph Wallpaper Editor
- **Description:** Visual, drag-and-drop authoring environment within the Gallery UI for composing multi-layered wallpapers with particle emitters, sprites, audio-reactive bars, and 2D physics without writing code.
- **Source:** Competitor Analysis (`GAP-02`), Step 1 Baseline (`OFF-009`).
- **Classification:** `FUTURE-OPPORTUNITY` (Priority: `P3`)
- **Intended Users:** Digital artists, graphic designers, and casual creators.
- **Expected Value:** Enables non-programmers to create professional interactive wallpapers.
- **Technical Feasibility:** `LOW` to `MEDIUM` — Requires building a 2D canvas editor (PixiJS / WebGL / Three.js node graph) within Tauri frontend.
- **Security Risks:** Low; sandboxed visual node graph compiles into declarative JSON manifests.
- **Resource Implications:** Heavy memory/CPU footprint during authoring session; 0 MB when UI is closed.
- **Dependencies:** `KW-NF-002` (Tauri UI), `KW-F-033` (Manifest V1), `V1-EXCL-001`.
- **Maintenance Burden:** Very High (estimated 25,000+ lines of frontend TypeScript).
- **Candidate Release:** `V2.0+ CANDIDATE`
- **Acceptance Criteria if Approved:** User can create, layer, animate, and export a functional particle wallpaper without code.
- **Approval Status:** `AWAITING APPROVAL`

#### `OFF-010`: RGB Hardware Peripheral Lighting Synchronization
- **Description:** Synchronize wallpaper color palettes and audio-reactive frequency beats with RGB gaming peripherals (Razer Chroma, Corsair iCUE, Logitech G-LIGHTSYNC, OpenRGB).
- **Source:** Competitor Analysis (`OPP-B04`), Step 1 Baseline (`OFF-010`).
- **Classification:** `FUTURE-OPPORTUNITY` (Priority: `P3`)
- **Intended Users:** PC gamers with full RGB battle-station setups.
- **Expected Value:** Immersive room-scale lighting synchronized with desktop video and music.
- **Technical Feasibility:** `MEDIUM` — Via OpenRGB C-API SDK or vendor-specific REST/C++ APIs.
- **Security Risks:** Third-party vendor DLL crashes or USB communication lockups.
- **Resource Implications:** Low; sending RGB packets over USB consumes <0.2% CPU.
- **Dependencies:** `KW-F-039` (WASAPI FFT), `OFF-001` (Color Extraction).
- **Maintenance Burden:** High; constant vendor firmware and SDK breaking changes.
- **Candidate Release:** `V2.0+ CANDIDATE`
- **Acceptance Criteria if Approved:** Peripherals smoothly reflect dominant wallpaper color with <50ms latency.
- **Approval Status:** `AWAITING APPROVAL`

---

### 2.3 Research & Exploratory Horizons (Target: Prototyping Spikes)

#### `OFF-011`: Local AI Wallpaper Generation (Stable Diffusion ONNX)
- **Description:** On-demand generation of custom desktop wallpapers using local DirectML/ONNX acceleration based on user text prompts.
- **Source:** Step 1 Baseline (`OFF-011`), Competitor Analysis (`OPP-C03`).
- **Classification:** `EXPLORATION ONLY` (Priority: `P3`)
- **Feasibility & Risk:** Extremely high VRAM/RAM requirement (4–8 GB); models require multi-gigabyte downloads. Violates V1.0 lightweight philosophy.
- **Candidate Release:** `EXPLORATION ONLY` (Phase 2 Prototyping Spike)
- **Approval Status:** `AWAITING APPROVAL`

#### `OFF-012`: AI Video Frame Rate Interpolation (RIFE DirectML)
- **Description:** Real-time or pre-computed neural network video interpolation (e.g. converting 24/30 FPS video to butter-smooth 60 FPS desktop presentation).
- **Source:** Step 1 Baseline (`OFF-012`), Competitor Analysis (`OPP-C02`).
- **Classification:** `EXPLORATION ONLY` (Priority: `P3`)
- **Feasibility & Risk:** High continuous GPU shader utilization; introduces severe battery and gaming thermal contention.
- **Candidate Release:** `EXPLORATION ONLY`
- **Approval Status:** `AWAITING APPROVAL`

#### `OPP-B01`: Native DirectX 12 Compute Shader Engine
- **Description:** A dedicated, ultra-low-overhead pure-Rust Direct3D 12 compute shader wallpaper player bypassing WebView2 entirely for custom HLSL shaders.
- **Source:** Competitor Analysis (`OPP-B01`).
- **Classification:** `FUTURE-OPPORTUNITY` (Priority: `P3`)
- **Candidate Release:** `V2.0+ CANDIDATE`
- **Approval Status:** `AWAITING APPROVAL`

#### `OPP-C05`: Linux (Wayland) & macOS Operating System Ports
- **Description:** Native live wallpaper daemons targeting Linux (`wlr-layer-shell` on Sway/Hyprland) and macOS (`CGSSetWindowLevel` / Metal).
- **Source:** Competitor Analysis (`OPP-C05`).
- **Classification:** `FUTURE-OPPORTUNITY` (Priority: `P3`)
- **Candidate Release:** `V2.0+ CANDIDATE`
- **Approval Status:** `AWAITING APPROVAL`

