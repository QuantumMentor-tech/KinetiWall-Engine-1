# KinetiWall — Formal Risk Register & Assessment Matrix
**Document ID:** `KW-QAR-006`  
**Deliverable:** Deliverable F — Formal Risk Register  
**Phase:** 1 — Project Research, Requirements & Feasibility  
**Step:** 6 of 90 — Acceptance Criteria, Success Metrics & Risk Register  
**Status:** COMPLETE (Auditable Risk Baseline)  
**Authority:** Technical Program Lead & Risk Management Lead  
**Date:** 2026-11-10  

---

## 1. Risk Scoring Methodology & Bands

Every system risk is evaluated on a standardized 5x5 Inherent and Residual Risk Matrix:

$$\text{Risk Score} = \text{Likelihood (1–5)} \times \text{Impact (1–5)}$$

### 1.1 Likelihood Scale
- **1 — Rare:** Unlikely to occur under ordinary supported conditions.
- **2 — Unlikely:** Possible in obscure edge cases but not expected.
- **3 — Possible:** Credible occurrence under relevant normal operating conditions.
- **4 — Likely:** Expected to arise in specific configurations without active mitigations.
- **5 — Almost Certain:** Expected to occur frequently or already observed repeatedly in testing.

### 1.2 Impact Scale
- **1 — Negligible:** Minor cosmetic inconvenience with immediate, automatic recovery.
- **2 — Minor:** Limited functional degradation; non-essential workflow workaround available.
- **3 — Moderate:** Material degradation to a secondary feature; does not crash engine.
- **4 — Major:** Failure of a core capability (video/web playback, pause engine, multi-monitor).
- **5 — Critical:** System crash, security vulnerability, licensing violation, or corrupted desktop shell.

### 1.3 Risk Severity Rating Bands
- **Low (1–4):** Acceptable with routine monitoring.
- **Moderate (5–9):** Requires standard engineering prevention and automated testing.
- **High (10–16):** Substantial threat requiring dedicated prototype spike and active architectural controls.
- **Critical (17–25):** Severe existential threat requiring executive escalation, proof-of-concept validation, and release-blocking status.

---

## 2. Exhaustive 20-Risk Register (`KW-R-001` through `KW-R-020`)

The register below documents the 20 minimum mandatory risks across all technical, platform, security, and project domains:

### `KW-R-001`: Undocumented Windows Desktop Integration Fragility
- **Title:** Windows Shell Desktop Attachment Incompatibility & Detachment
- **Description:** Message `0x052C` to `Progman` to spawn `WorkerW` is an undocumented Win32 shell technique that can alter behavior across Windows updates.
- **Category:** Windows Compatibility & Technical Feasibility
- **Cause:** Microsoft modifies internal desktop window hierarchy in major updates (e.g., Windows 11 24H2 detaching WorkerW when desktop icons are hidden).
- **Potential Consequence:** Wallpaper window detaches, turns solid black, or renders in front of desktop icons, blocking user interaction.
- **Affected Requirements:** `KW-F-001`, `KW-F-002`, `KW-F-004`
- **Affected Components:** Desktop Shell Attachment Engine, `kinetiwall-core.exe`
- **Inherent Likelihood:** 4 (Likely) | **Inherent Impact:** 5 (Critical) | **Inherent Score:** 20 (Critical)
- **Existing Controls:** Dual-Mode attachment algorithm defined in `DEC-PLT-001`.
- **Mitigation Strategy:** Automated runtime detection of icon visibility; fallback to direct `Progman` child attachment beneath `SHELLDLL_DefView` when icons are hidden.
- **Prevention Measures:** Isolate attachment routines into dedicated crate; avoid DLL injection or hooking.
- **Detection Method:** Continuous window hierarchy monitoring thread verifying parent-child relationships.
- **Contingency / Fallback:** Revert desktop to native static wallpaper and notify user of compatibility restriction.
- **Risk Owner:** Principal Windows Platform Architect
- **Validation Evidence:** Prototype Gate `VG-001` across Windows 11 24H2 and Windows 10 22H2 VMs.
- **Residual Likelihood:** 2 (Unlikely) | **Residual Impact:** 4 (Major) | **Residual Score:** 8 (Moderate)
- **Current Status:** `ACTIVE (MITIGATION PLANNED)`
- **Review Milestone:** Milestone 2 (Phase 2 Prototype Spikes)
- **Source / Rationale:** Step 4 Platform Risk Register (`DR-001`)

---

### `KW-R-002`: Explorer Shell Crash & Desktop Re-Attachment Failure
- **Title:** Unhandled `explorer.exe` Crash Leaving Desktop in Inconsistent State
- **Description:** When Windows Explorer crashes or restarts, previously spawned `WorkerW` and desktop window handles become invalid.
- **Category:** Reliability & Recovery
- **Cause:** Windows shell updates, third-party shell extensions, or user manual task termination.
- **Potential Consequence:** Live wallpaper crashes, leaves orphan top-level windows, or fails to re-appear.
- **Affected Requirements:** `KW-F-003`, `KW-F-007`
- **Affected Components:** Core Controller Daemon, Windows Event Pump
- **Inherent Likelihood:** 4 (Likely) | **Inherent Impact:** 4 (Major) | **Inherent Score:** 16 (High)
- **Existing Controls:** Architecture listens for `TaskbarCreated` registered window message.
- **Mitigation Strategy:** Re-execute attachment handshake within ≤2.5s of message receipt.
- **Prevention Measures:** Validate all window handles (`IsWindow`) before dispatching win32 commands.
- **Detection Method:** Window message listener catching `TaskbarCreated`.
- **Contingency / Fallback:** Wait 3 seconds, retry attachment up to 3 times before graceful sleep.
- **Risk Owner:** Senior Systems Reliability Engineer
- **Validation Evidence:** Automated fault injection test `REL-01` (30 iterations).
- **Residual Likelihood:** 1 (Rare) | **Residual Impact:** 3 (Moderate) | **Residual Score:** 3 (Low)
- **Current Status:** `ACTIVE (CONTROLLED)`
- **Review Milestone:** Milestone 3 (Phase 3 Core Foundation)
- **Source / Rationale:** Step 1 Requirements (`KW-F-003`)

---

### `KW-R-003`: False Fullscreen or Occlusion Detection
- **Title:** Misclassification of Foreground Games, Overlays & Borderless Windows
- **Description:** Foreground window detection misidentifies transparent overlays (Discord, GeForce Experience) as fullscreen games or misses borderless windowed games.
- **Category:** Performance & User Experience
- **Cause:** Complex Win32 window styles (`WS_EX_LAYERED`, `WS_POPUP`, multi-monitor spanning).
- **Potential Consequence:** Wallpaper continues playing behind 3D games (wasting GPU/battery) or pauses erroneously during normal desktop work.
- **Affected Requirements:** `KW-F-026`, `KW-F-027`, `KW-NF-004`
- **Affected Components:** Occlusion Engine, WinEvent Hook Listener
- **Inherent Likelihood:** 4 (Likely) | **Inherent Impact:** 4 (Major) | **Inherent Score:** 16 (High)
- **Existing Controls:** Event-driven `SetWinEventHook` listening to `EVENT_SYSTEM_FOREGROUND`.
- **Mitigation Strategy:** Combine window rect bounds check with `GetWindowLongPtrW` checking `WS_VISIBLE` and non-transparent styles.
- **Prevention Measures:** Ignore known overlay windows via class name whitelist; verify window covers full display rect.
- **Detection Method:** Continuous ETW trace logging window transition events.
- **Contingency / Fallback:** Provide user manual pause toggle and per-app override rules (`OFF-007`).
- **Risk Owner:** Senior Performance QA Engineer
- **Validation Evidence:** CapFrameX comparative gaming benchmark `BENCH-03`.
- **Residual Likelihood:** 2 (Unlikely) | **Residual Impact:** 2 (Minor) | **Residual Score:** 4 (Low)
- **Current Status:** `ACTIVE (CONTROLLED)`
- **Review Milestone:** Milestone 3 (Phase 3 Event Engine)
- **Source / Rationale:** Step 2 Technical Research (`TR-005`)

---

### `KW-R-004`: Incomplete Resource Release on Playback Pause
- **Title:** Video/Web Players Retaining GPU Cycles or Thread Activity While Paused
- **Description:** A player halts visible rendering on screen but continues internal timer loops, audio capture, or background shader passes.
- **Category:** Performance & Power Governance
- **Cause:** libmpv render loop not completely frozen; WebView2 JavaScript timers firing at full speed.
- **Potential Consequence:** Battery drain on laptops; GPU contention with AAA gaming titles.
- **Affected Requirements:** `KW-F-010`, `KW-F-030`, `KW-NF-005`, `KW-NF-007`
- **Affected Components:** `kinetiwall-player-video.exe`, `kinetiwall-player-web.exe`
- **Inherent Likelihood:** 3 (Possible) | **Inherent Impact:** 5 (Critical) | **Inherent Score:** 15 (High)
- **Existing Controls:** Explicit mpv `pause=yes` command and WebView2 `put_IsVisible(FALSE)`.
- **Mitigation Strategy:** Freeze Direct3D swapchain presentation immediately; throttle JavaScript timers to 0 Hz.
- **Prevention Measures:** Verify 0.00% GPU utilization in continuous automated benchmarks.
- **Detection Method:** GPU-Z / NVAPI performance counter sampling during paused state.
- **Contingency / Fallback:** Drop player process to Windows `BelowNormal` priority or static fallback blitting.
- **Risk Owner:** Lead Media Pipeline Engineer
- **Validation Evidence:** Performance Benchmark `BENCH-03` and Validation Gate `VG-004`.
- **Residual Likelihood:** 1 (Rare) | **Residual Impact:** 3 (Moderate) | **Residual Score:** 3 (Low)
- **Current Status:** `ACTIVE (CONTROLLED)`
- **Review Milestone:** Milestone 4 (Phase 4 Player Subsystem)
- **Source / Rationale:** Step 1 Requirements (`KW-F-010`)

---

### `KW-R-005`: Graphics Driver & Hardware Video Decoding Variability
- **Title:** Codec Incompatibility & GPU Driver Crashes Across Hardware Vendors
- **Description:** Hardware acceleration (`D3D11VA`) behaves inconsistently across Intel, NVIDIA, and AMD driver versions or fails on corrupt media files.
- **Category:** Third-Party Dependencies & Platform Compatibility
- **Cause:** Differences in driver DXGI implementation, hardware decode engine limits, or missing codec profiles.
- **Potential Consequence:** Player crashes, produces artifacted/green video frames, or triggers high CPU usage.
- **Affected Requirements:** `KW-F-009`, `KW-F-013`, `KW-NF-003`, `KW-NF-009`
- **Affected Components:** Video Player Subsystem, `libmpv-2.dll`
- **Inherent Likelihood:** 4 (Likely) | **Inherent Impact:** 4 (Major) | **Inherent Score:** 16 (High)
- **Existing Controls:** `libmpv` FFmpeg-based multi-format decoding engine.
- **Mitigation Strategy:** Configure graceful fallback: attempt `hwdec=d3d11va`; if initialization fails, fall back to `hwdec=auto-safe` or software decoding.
- **Prevention Measures:** Extensive fuzzing of video container parser; automated corrupt file isolation.
- **Detection Method:** Capture libmpv log callbacks (`MPV_LOG_LEVEL_WARN`).
- **Contingency / Fallback:** Revert to static preview image and log user-friendly error notification.
- **Risk Owner:** Principal Media Systems Architect
- **Validation Evidence:** Multi-GPU test suite across NVIDIA/AMD/Intel in Validation Gate `VG-003`.
- **Residual Likelihood:** 2 (Unlikely) | **Residual Impact:** 3 (Moderate) | **Residual Score:** 6 (Moderate)
- **Current Status:** `ACTIVE (MITIGATION PLANNED)`
- **Review Milestone:** Milestone 4 (Phase 4 Player Subsystem)
- **Source / Rationale:** Step 2 Technical Research (`TR-004`)

---

### `KW-R-006`: WebView2 Runtime Availability & Version Fragmentation
- **Title:** Missing or Incompatible Microsoft Edge WebView2 Evergreen Runtime
- **Description:** Target system lacks WebView2 Evergreen Runtime or has a corrupted runtime installation, preventing web wallpaper execution.
- **Category:** Third-Party Dependencies & Offline Distribution
- **Cause:** Air-gapped Windows 10 installations or unmanaged enterprise workstations without Microsoft Edge.
- **Potential Consequence:** Web wallpapers fail to initialize; application crashes on launch.
- **Affected Requirements:** `KW-F-014`, `KW-NF-018`, `KW-NF-023`
- **Affected Components:** Web Player Subsystem, Offline Installer
- **Inherent Likelihood:** 3 (Possible) | **Inherent Impact:** 4 (Major) | **Inherent Score:** 12 (High)
- **Existing Controls:** Platform policy verifies WebView2 Evergreen presence.
- **Mitigation Strategy:** Installer checks for WebView2 registry key (`HKLM\SOFTWARE\WOW6432Node\Microsoft\EdgeUpdate\Clients\{F3017226-6E2A-4216-BCAF-E61007305804}`).
- **Prevention Measures:** Provide clear offline redistributable documentation and installer warning if absent.
- **Detection Method:** Pre-flight check during `kinetiwall-core.exe` initialization.
- **Contingency / Fallback:** Disable web wallpaper options gracefully in UI while keeping video wallpapers 100% operational.
- **Risk Owner:** Technical Program Manager
- **Validation Evidence:** Offline air-gapped test scenario `CT-018` and Validation Gate `VG-008`.
- **Residual Likelihood:** 1 (Rare) | **Residual Impact:** 3 (Moderate) | **Residual Score:** 3 (Low)
- **Current Status:** `ACTIVE (CONTROLLED)`
- **Review Milestone:** Milestone 2 (Phase 2 Prototyping)
- **Source / Rationale:** Step 4 Platform Policy (`DEC-PLT-003`)

---

### `KW-R-007`: Unsafe Web Wallpaper Content & Sandboxing Escape
- **Title:** Malicious JavaScript or Archive Exploits in User-Imported Wallpapers
- **Description:** A packaged web wallpaper contains cross-origin exfiltration scripts, local path traversal attacks, or runaway memory loops.
- **Category:** Security & Privacy
- **Cause:** Untrusted community-downloaded wallpaper packages.
- **Potential Consequence:** Data exfiltration, local file access, or system memory exhaustion.
- **Affected Requirements:** `KW-F-016`, `KW-F-018`, `KW-NF-014`, `KW-NF-016`
- **Affected Components:** Web Player Subsystem, Archive Extraction Engine
- **Inherent Likelihood:** 3 (Possible) | **Inherent Impact:** 5 (Critical) | **Inherent Score:** 15 (High)
- **Existing Controls:** Virtual Host Directory Mapping (`https://kinetiwall.local/`) with CORS denial.
- **Mitigation Strategy:** Unconditional denial of camera/mic permissions, blocked popups, Zip-Slip canonical path checks, and 500 MB RAM Job Object quotas.
- **Prevention Measures:** Enforce Content Security Policy (CSP) and disable `file://` URL schemes.
- **Detection Method:** Automated security penetration test suite `SEC-01` through `SEC-07`.
- **Contingency / Fallback:** Terminate offending player process immediately; disable auto-restart for that wallpaper.
- **Risk Owner:** Senior Security Research Analyst
- **Validation Evidence:** Validation Gate `VG-006` security penetration audit.
- **Residual Likelihood:** 1 (Rare) | **Residual Impact:** 4 (Major) | **Residual Score:** 4 (Low)
- **Current Status:** `ACTIVE (CONTROLLED)`
- **Review Milestone:** Milestone 4 (Phase 4 Web Player)
- **Source / Rationale:** Step 2 Research (`TR-006`)

---

### `KW-R-008`: IPC Authorization & Named Pipe Hijacking
- **Title:** Unauthorized Local IPC Access via Insecure Named Pipe Security
- **Description:** Unprivileged local processes or secondary user sessions connect to the KinetiWall Named Pipe to control engine playback or inject commands.
- **Category:** Security & Architecture
- **Cause:** Default, overly permissive Windows Named Pipe security descriptors (`NULL` DACL).
- **Potential Consequence:** Local privilege escalation or unauthorized desktop manipulation.
- **Affected Requirements:** `KW-NF-003`, `KW-NF-012`, `KW-NF-013`
- **Affected Components:** Named Pipe IPC Server (`kinetiwall-core.exe`)
- **Inherent Likelihood:** 3 (Possible) | **Inherent Impact:** 5 (Critical) | **Inherent Score:** 15 (High)
- **Existing Controls:** Win32 DACL architecture binding pipe to interactive logon SID.
- **Mitigation Strategy:** Configure explicit security descriptor granting `FILE_GENERIC_READ | FILE_GENERIC_WRITE` strictly to current user token.
- **Prevention Measures:** Enforce strict Serde JSON schema validation on all incoming IPC payloads.
- **Detection Method:** Penetration test attempting connection from secondary user account (`SEC-01`).
- **Contingency / Fallback:** Reject connection with `ERROR_ACCESS_DENIED` and log security alert.
- **Risk Owner:** Systems Security Architect
- **Validation Evidence:** Automated security test `SEC-01` and `SEC-02`.
- **Residual Likelihood:** 1 (Rare) | **Residual Impact:** 3 (Moderate) | **Residual Score:** 3 (Low)
- **Current Status:** `ACTIVE (CONTROLLED)`
- **Review Milestone:** Milestone 3 (Phase 3 IPC Foundation)
- **Source / Rationale:** Step 1 Requirements (`KW-NF-012`)

---

### `KW-R-009`: Process Lifecycle & Orphaned Player Accumulation
- **Title:** Wallpaper Players Outliving Core Controller or Accumulating in Memory
- **Description:** Unexpected termination of `kinetiwall-core.exe` leaves child video/web player processes running indefinitely in the background.
- **Category:** Reliability & Resource Management
- **Cause:** Abrupt daemon crash, unhandled panic, or Windows shutdown race condition.
- **Potential Consequence:** Resource leaks, multiple conflicting players rendering to desktop, high RAM waste.
- **Affected Requirements:** `KW-F-006`, `KW-NF-010`
- **Affected Components:** Process Supervisor, Windows Job Objects
- **Inherent Likelihood:** 4 (Likely) | **Inherent Impact:** 4 (Major) | **Inherent Score:** 16 (High)
- **Existing Controls:** Binding all child processes to Windows Job Objects.
- **Mitigation Strategy:** Configure `JOB_OBJECT_LIMIT_KILL_ON_JOB_CLOSE` on job object handle.
- **Prevention Measures:** Kernel-enforced termination ensures child processes die the instant the job handle closes.
- **Detection Method:** 50-run automated fault-injection test `REL-03`.
- **Contingency / Fallback:** Watchdog process heartbeat verification.
- **Risk Owner:** Senior Systems Reliability Engineer
- **Validation Evidence:** Validation Gate `VG-002`.
- **Residual Likelihood:** 1 (Rare) | **Residual Impact:** 2 (Minor) | **Residual Score:** 2 (Low)
- **Current Status:** `ACTIVE (CONTROLLED)`
- **Review Milestone:** Milestone 3 (Phase 3 Job Objects)
- **Source / Rationale:** Step 2 Technical Research (`TR-002`)

---

### `KW-R-010`: Multi-Monitor Topology & Coordinate Inconsistency
- **Title:** Visual Tearing, Window Misplacement & DPI Blurring Across Multiple Displays
- **Description:** Display hotplugging, mixed DPI scaling (100% vs 200%), or display rotation causes wallpaper windows to misalign or scale incorrectly.
- **Category:** Architecture & User Experience
- **Cause:** Negative virtual screen coordinates, non-linear DPI awareness, or asynchronous display messages.
- **Potential Consequence:** Wallpapers stretched across wrong screens, visual seams, blurred rendering.
- **Affected Requirements:** `KW-F-020` through `KW-F-025`
- **Affected Components:** Multi-Monitor Topology Manager, DXGI Swapchain Presenter
- **Inherent Likelihood:** 4 (Likely) | **Inherent Impact:** 3 (Moderate) | **Inherent Score:** 12 (High)
- **Existing Controls:** Win32 Per-Monitor V2 DPI awareness manifest configuration.
- **Mitigation Strategy:** Re-query `EnumDisplayMonitors` on every `WM_DISPLAYCHANGE` and update rendering viewports dynamically.
- **Prevention Measures:** Handle negative coordinates explicitly using `RECT` offset math.
- **Detection Method:** Automated display mode toggle test harness `REL-05`.
- **Contingency / Fallback:** Fallback to single primary monitor mode if virtual desktop bounds cannot be resolved.
- **Risk Owner:** Senior Graphics & Platform Engineer
- **Validation Evidence:** Multi-Monitor Test Suite `CT-006` through `CT-011`.
- **Residual Likelihood:** 2 (Unlikely) | **Residual Impact:** 2 (Minor) | **Residual Score:** 4 (Low)
- **Current Status:** `ACTIVE (CONTROLLED)`
- **Review Milestone:** Milestone 6 (Phase 6 Multi-Monitor)
- **Source / Rationale:** Step 1 Requirements (`KW-F-020`)

---

### `KW-R-011`: Audio Capture Failure & Surveillance Misconceptions
- **Title:** WASAPI Audio Loopback Capture Failure or Privacy Concerns
- **Description:** Audio-reactive visualizer fails on default audio device changes or creates user concern regarding audio recording/surveillance.
- **Category:** Privacy & Multimedia
- **Cause:** USB headphone unplugging (`WM_DEVICECHANGE`) or uncalibrated FFT frequency calculation.
- **Potential Consequence:** Audio visualizer freezes, produces audible stutter, or triggers user privacy backlash.
- **Affected Requirements:** `KW-F-039` through `KW-F-042`, `KW-NF-014`, `KW-NG-005`
- **Affected Components:** Audio Capture Subsystem, Pure-Rust FFT Worker
- **Inherent Likelihood:** 3 (Possible) | **Inherent Impact:** 4 (Major) | **Inherent Score:** 12 (High)
- **Existing Controls:** Explicit user opt-in toggle; volatile-only ring buffers (`KW-NG-005`).
- **Mitigation Strategy:** Listen to `IMMNotificationClient` audio device change events; auto-reconnect capture client in <500ms.
- **Prevention Measures:** Never serialize audio data to disk; zero disk write calls in audio crate.
- **Detection Method:** Process Monitor file write audit confirming 0 bytes written (`SEC-09`).
- **Contingency / Fallback:** Stop audio capture thread gracefully; visualizer displays idle animation.
- **Risk Owner:** Principal Audio Systems Engineer
- **Validation Evidence:** Validation Gate `VG-007` and Security Test `SEC-09`.
- **Residual Likelihood:** 1 (Rare) | **Residual Impact:** 2 (Minor) | **Residual Score:** 2 (Low)
- **Current Status:** `ACTIVE (CONTROLLED)`
- **Review Milestone:** Milestone 4 (Phase 4 Audio Engine)
- **Source / Rationale:** Step 1 Requirements (`KW-NG-005`)

---

### `KW-R-012`: Media Library Licensing & LGPL Redistribution Liability
- **Title:** Copyleft License Infection or LGPL v2.1+ Compliance Violation
- **Description:** Static linking or improper distribution of `libmpv` or FFmpeg components violates LGPL v2.1+, creating legal liability.
- **Category:** Licensing & Legal Compliance
- **Cause:** Build configuration accidentally statically linking `libmpv` or omitting required source attribution notices.
- **Potential Consequence:** Copyright infringement claims, mandatory open-sourcing of proprietary modules, or distribution injunctions.
- **Affected Requirements:** `KW-F-046`, `DEC-PLT-004`
- **Affected Components:** Video Player Build Pipeline, Packaging Installer
- **Inherent Likelihood:** 3 (Possible) | **Inherent Impact:** 5 (Critical) | **Inherent Score:** 15 (High)
- **Existing Controls:** Strict architecture policy mandating dynamic linking (`DEC-PLT-004`).
- **Mitigation Strategy:** Dynamically link `libmpv-2.dll`; bundle full LGPL v2.1 text, copyright notices, and custom DLL replacement instructions.
- **Prevention Measures:** Automated PE header audit in CI pipeline verifying dynamic import table.
- **Detection Method:** Validation Gate `VG-010` DLL replacement verification test.
- **Contingency / Fallback:** Replace libmpv with Windows Media Foundation (WMF) if licensing terms cannot be met.
- **Risk Owner:** Open Source Compliance Lead
- **Validation Evidence:** Validation Gate `VG-010` and Release Gate `RAC-06`.
- **Residual Likelihood:** 1 (Rare) | **Residual Impact:** 4 (Major) | **Residual Score:** 4 (Low)
- **Current Status:** `ACTIVE (CONTROLLED)`
- **Review Milestone:** Milestone 2 (Phase 2 Prototyping)
- **Source / Rationale:** Step 4 Platform Decisions (`DEC-PLT-004`)

---

### `KW-R-013`: Offline Installation Dependencies & Conflict
- **Title:** Installer Failure on Air-Gapped Machines Due to Missing Online Packages
- **Description:** Installer fails on clean, offline Windows installations because VC++ redistributables or WebView2 packages attempt internet downloads.
- **Category:** Packaging & Offline Distribution
- **Cause:** Relying on online web-bootstrapper installers instead of self-contained offline packages.
- **Potential Consequence:** Complete installation failure for enterprise or privacy-conscious offline users.
- **Affected Requirements:** `KW-F-043`, `KW-NF-006`, `KW-NG-003`
- **Affected Components:** Standalone Installer (`NSIS` / `WiX`)
- **Inherent Likelihood:** 3 (Possible) | **Inherent Impact:** 5 (Critical) | **Inherent Score:** 15 (High)
- **Existing Controls:** Offline installation requirement defined in `KW-NG-003`.
- **Mitigation Strategy:** Build self-contained offline installer bundling pre-compiled engine binaries, local `libmpv-2.dll`, and offline runtime checks.
- **Prevention Measures:** Test installer on air-gapped clean Windows virtual machine in CI.
- **Detection Method:** Automated VM offline installation test `CT-018`.
- **Contingency / Fallback:** Provide portable `.zip` release alongside standard installer.
- **Risk Owner:** DevOps & Release Engineering Lead
- **Validation Evidence:** Validation Gate `VG-008` and Acceptance Test `KW-AC-022`.
- **Residual Likelihood:** 1 (Rare) | **Residual Impact:** 3 (Moderate) | **Residual Score:** 3 (Low)
- **Current Status:** `ACTIVE (CONTROLLED)`
- **Review Milestone:** Milestone 7 (Phase 7 Packaging)
- **Source / Rationale:** Step 1 Requirements (`KW-NG-003`)

---

### `KW-R-014`: ARM64 Platform Compatibility & Emulation Degradation
- **Title:** Unacceptable Performance or Crash Under Windows 11 ARM64 Prism Emulation
- **Description:** While x86-64 is the primary launch architecture, running under ARM64 emulation (Snapdragon X Elite) might trigger high CPU load or D3D11VA failures.
- **Category:** Windows Compatibility & Architecture
- **Cause:** Hardware video decode acceleration translation issues under Prism emulation.
- **Potential Consequence:** Video stuttering or excessive battery drain on Copilot+ ARM64 PCs.
- **Affected Requirements:** `KW-NF-018`, `DEC-PLT-002`
- **Affected Components:** Video Player Engine, Direct3D 11 Pipeline
- **Inherent Likelihood:** 3 (Possible) | **Inherent Impact:** 3 (Moderate) | **Inherent Score:** 9 (Moderate)
- **Existing Controls:** Step 4 Platform Policy classifies ARM64 as Tier B (Supported via Prism Emulation).
- **Mitigation Strategy:** Benchmark x86-64 binary on Windows 11 24H2 ARM64; verify D3D11VA hardware decode offload.
- **Prevention Measures:** Structure Rust workspace for eventual native ARM64 compilation in post-V1 release.
- **Detection Method:** ARM64 virtualized / physical hardware test run.
- **Contingency / Fallback:** Document ARM64 emulation status and limits in release notes.
- **Risk Owner:** Windows Compatibility Lead
- **Validation Evidence:** Platform Test Scenario `CT-016` (ARM64 Prism Emulation).
- **Residual Likelihood:** 2 (Unlikely) | **Residual Impact:** 2 (Minor) | **Residual Score:** 4 (Low)
- **Current Status:** `ACTIVE (MONITORED)`
- **Review Milestone:** Milestone 6 (Phase 6 Platform Gate)
- **Source / Rationale:** Step 4 Platform Support (`DEC-PLT-002`)

---

### `KW-R-015`: Scope Inflation & Architectural Over-Complexity
- **Title:** Premature Implementation of Complex 3D Executable & Cloud Features
- **Description:** Attempting to build standalone Win32 game reparenting, scene-graph editors, and cloud workshops in V1.0 delays the core engine release.
- **Category:** Technical Feasibility & Project Delivery
- **Cause:** Treating exploratory research ideas as mandatory initial release requirements.
- **Potential Consequence:** Multi-month schedule slip, brittle architecture, and buggy core features.
- **Affected Requirements:** `KW-F-019`, `OFF-008`, `OFF-009`, `V1-EXCL-001`..`010`
- **Affected Components:** Entire Project Architecture
- **Inherent Likelihood:** 4 (Likely) | **Inherent Impact:** 4 (Major) | **Inherent Score:** 16 (High)
- **Existing Controls:** Rigorous release boundaries defined in Step 5 Release Scope (`KW-SCP-001`).
- **Mitigation Strategy:** Deliver 3D in V1.0 via WebGL in WebView2; strictly defer standalone `.exe` reparenting to V2.0+; enforce Phase 1–7 gates.
- **Prevention Measures:** Prohibit authoring features not explicitly scheduled in V1.0 scope baseline.
- **Detection Method:** Bi-weekly scope audit comparing git commits against `COMPLETE_FEATURE_INVENTORY.md`.
- **Contingency / Fallback:** Re-scope non-essential items to V1.x upon first sign of milestone slippage.
- **Risk Owner:** Principal Software Architect & Program Lead
- **Validation Evidence:** Step 5 Release Scope Proposal (`KW-SCP-001`).
- **Residual Likelihood:** 1 (Rare) | **Residual Impact:** 3 (Moderate) | **Residual Score:** 3 (Low)
- **Current Status:** `ACTIVE (CONTROLLED)`
- **Review Milestone:** Phase 1 Governance Sign-off
- **Source / Rationale:** Step 5 Release Scope Proposal

---

### `KW-R-016`: Inadequate Automated Regression Test Coverage
- **Title:** Latent Regressions in Desktop Attachment or IPC Escaping Detection
- **Description:** Modifications to core controller logic break desktop icon interaction or crash recovery without being caught by CI.
- **Category:** Testing & Validation
- **Cause:** Relying solely on manual "smoke testing" instead of automated headless integration tests.
- **Potential Consequence:** Broken releases shipped to end users; loss of user trust.
- **Affected Requirements:** `KW-NF-027`
- **Affected Components:** CI/CD Build Pipeline, Test Harness
- **Inherent Likelihood:** 3 (Possible) | **Inherent Impact:** 4 (Major) | **Inherent Score:** 12 (High)
- **Existing Controls:** Modular Cargo workspace with headless mock testing architecture.
- **Mitigation Strategy:** Maintain minimum 75% unit test coverage; implement automated VM integration tests for shell attachment.
- **Prevention Measures:** Block PR merges if automated test suite fails or coverage drops.
- **Detection Method:** Automated `cargo test` and `cargo-tarpaulin` reporting in CI.
- **Contingency / Fallback:** Mandatory manual QA release checklist execution (`RAC-01`..`15`) prior to any release tag.
- **Risk Owner:** Senior QA Architect
- **Validation Evidence:** Automated Test Coverage Baseline `KW-MET-030`.
- **Residual Likelihood:** 1 (Rare) | **Residual Impact:** 3 (Moderate) | **Residual Score:** 3 (Low)
- **Current Status:** `ACTIVE (CONTROLLED)`
- **Review Milestone:** Phase 6 QA Gate
- **Source / Rationale:** Step 1 Requirements (`KW-NF-027`)

---

### `KW-R-017`: Configuration & Local Library Corruption
- **Title:** Loss of User Settings or Library Manifests on Power Loss or Crash
- **Description:** Abrupt PC shutdown or crash during configuration update corrupts `config.json`, wiping custom wallpaper settings.
- **Category:** Data Integrity & Reliability
- **Cause:** Non-atomic file write routines (`std::fs::write` directly overwriting destination).
- **Potential Consequence:** User configurations reset to defaults; corrupted library state.
- **Affected Requirements:** `KW-F-037`
- **Affected Components:** Configuration Storage Engine
- **Inherent Likelihood:** 3 (Possible) | **Inherent Impact:** 3 (Moderate) | **Inherent Score:** 9 (Moderate)
- **Existing Controls:** Atomic temporary file write and backup architecture (`config.json.bak`).
- **Mitigation Strategy:** Write configuration to `.tmp` file, flush to disk via `sync_all()`, atomically rename over target, maintain rolling `.bak`.
- **Prevention Measures:** Auto-restore from backup if primary JSON fails parsing on startup.
- **Detection Method:** Fault injection test `REL-07` injecting corrupted bytes.
- **Contingency / Fallback:** Prompt user to restore previous profile if both files corrupted.
- **Risk Owner:** Senior Rust Systems Engineer
- **Validation Evidence:** Acceptance Test `KW-AC-020`.
- **Residual Likelihood:** 1 (Rare) | **Residual Impact:** 2 (Minor) | **Residual Score:** 2 (Low)
- **Current Status:** `ACTIVE (CONTROLLED)`
- **Review Milestone:** Milestone 4 (Phase 4 Library Engine)
- **Source / Rationale:** Step 1 Requirements (`KW-F-037`)

---

### `KW-R-018`: Unsubstantiated Marketing & Resource Claims
- **Title:** Misleading Users with Fabricated "Zero RAM" or "0% CPU" Claims
- **Description:** Marketing or release notes promise performance targets that fail on typical consumer hardware.
- **Category:** Performance & Project Integrity
- **Cause:** Measuring idle performance on a high-end 64-core workstation and generalizing to budget laptops.
- **Potential Consequence:** User dissatisfaction, negative reviews, credibility loss.
- **Affected Requirements:** `KW-NF-001` through `KW-NF-005`
- **Affected Components:** Documentation & Release Notes
- **Inherent Likelihood:** 3 (Possible) | **Inherent Impact:** 3 (Moderate) | **Inherent Score:** 9 (Moderate)
- **Existing Controls:** Strict empirical benchmarking protocols defined in `PERFORMANCE_BENCHMARK_PLAN.md`.
- **Mitigation Strategy:** Document exact hardware baselines, OS builds, and test durations alongside all published metrics.
- **Prevention Measures:** Label all unmeasured figures as `PROPOSED` or `NOT RUN`.
- **Detection Method:** Peer review audit comparing claims against WPR raw trace files.
- **Contingency / Fallback:** Publish conservative performance guidance and acknowledge hardware variances.
- **Risk Owner:** Technical Program Lead
- **Validation Evidence:** Documented benchmark runs in `BENCH-01` through `BENCH-05`.
- **Residual Likelihood:** 1 (Rare) | **Residual Impact:** 2 (Minor) | **Residual Score:** 2 (Low)
- **Current Status:** `ACTIVE (CONTROLLED)`
- **Review Milestone:** Milestone 7 (Phase 7 Release Gate)
- **Source / Rationale:** Step 6 Governance Standard

---

### `KW-R-019`: Unsafe Failure Handling & Desktop Inaccessibility
- **Title:** Engine Error Leaving Black Screen or Unclickable Windows Desktop
- **Description:** A severe player crash or unhandled hook error renders the desktop surface black and unresponsive, preventing user access to desktop files.
- **Category:** Architecture & User Safety
- **Cause:** Failure to restore desktop static wallpaper upon abnormal termination.
- **Potential Consequence:** User forced to reboot or restart `explorer.exe` manually to recover desktop access.
- **Affected Requirements:** `KW-F-004`, `KW-F-008`
- **Affected Components:** Core Controller Daemon, Windows Shell Hooks
- **Inherent Likelihood:** 3 (Possible) | **Inherent Impact:** 5 (Critical) | **Inherent Score:** 15 (High)
- **Existing Controls:** Panic hook (`std::panic::set_hook`) registered to restore desktop wallpaper.
- **Mitigation Strategy:** Install global Windows SEH / Rust panic hooks that execute `SystemParametersInfoW` even during unhandled panics.
- **Prevention Measures:** Emergency standalone recovery shortcut in Start Menu ("KinetiWall Reset Desktop").
- **Detection Method:** Crash simulation testing during Acceptance Test `KW-AC-004`.
- **Contingency / Fallback:** User presses `Ctrl+Shift+Esc`, terminates engine; static wallpaper restores.
- **Risk Owner:** Principal Systems Architect
- **Validation Evidence:** Acceptance Test `KW-AC-004`.
- **Residual Likelihood:** 1 (Rare) | **Residual Impact:** 3 (Moderate) | **Residual Score:** 3 (Low)
- **Current Status:** `ACTIVE (CONTROLLED)`
- **Review Milestone:** Milestone 3 (Phase 3 Core Foundation)
- **Source / Rationale:** Step 1 Requirements (`KW-F-004`)

---

### `KW-R-020`: Uncontrolled Scope Creep via AI / Community Additions
- **Title:** Premature Architecture Churn Caused by Experimental AI or RGB Features
- **Description:** Adding local AI image generation (Stable Diffusion) or RGB peripheral SDKs destabilizes core engine memory and installer footprints.
- **Category:** Release Management & Maintenance
- **Cause:** Introducing experimental features before core video/web stability is 100% locked down.
- **Potential Consequence:** Bloated 4 GB installer, memory leaks, crashing third-party vendor DLLs.
- **Affected Requirements:** `V1-EXCL-001` through `V1-EXCL-010`, `OFF-010`, `OFF-011`
- **Affected Components:** Build Pipeline, Core Controller
- **Inherent Likelihood:** 3 (Possible) | **Inherent Impact:** 4 (Major) | **Inherent Score:** 12 (High)
- **Existing Controls:** Step 5 Non-Goals Register (`KW-SCP-004`).
- **Mitigation Strategy:** Enforce strict quarantine: all AI and RGB features remain classified as `EXPLORATION ONLY` or `V2.0+ CANDIDATE`.
- **Prevention Measures:** Reject any PR introducing external AI model weights or vendor RGB SDKs into V1.0.
- **Detection Method:** Dependency manifest audits (`cargo tree` / `package.json`).
- **Contingency / Fallback:** Revert unauthorized dependency commits immediately.
- **Risk Owner:** Technical Program Lead
- **Validation Evidence:** Step 5 Non-Goals Register (`KW-SCP-004`).
- **Residual Likelihood:** 1 (Rare) | **Residual Impact:** 2 (Minor) | **Residual Score:** 2 (Low)
- **Current Status:** `ACTIVE (CONTROLLED)`
- **Review Milestone:** Phase 1 Governance Sign-off
- **Source / Rationale:** Step 5 Non-Goals Register

