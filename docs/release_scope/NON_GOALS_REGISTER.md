# KinetiWall — Non-Goals Register (Permanent Boundaries vs Release Exclusions)
**Document ID:** `KW-SCP-004`  
**Deliverable:** Deliverable D — Non-Goals Register  
**Phase:** 1 — Project Research, Requirements & Feasibility  
**Step:** 5 of 90 — Release Scope, Non-Goals & Feature Prioritization  
**Status:** COMPLETE (Auditable Boundary Baseline)  
**Authority:** Foundational Product Specification & Security Architecture  
**Date:** 2026-11-10  

---

## 1. Purpose & Separation Principle

A world-class systems application is defined as much by what it **refuses to do** as by what it implements. Scope discipline is essential to preserve KinetiWall's core value proposition: ultra-lightweight resource footprint, rock-solid stability, zero privacy invasion, and uncompromised user trust.

To avoid confusion, KinetiWall strictly separates:
1. **Permanent Project Anti-Goals:** Architectural, ethical, security, and behavioral boundaries that will **never** be violated in any version of KinetiWall.
2. **Temporary Release Exclusions (V1.0 Exclusions):** High-complexity, high-overhead, or speculative features deliberately excluded from Version 1.0 to ensure a flawless initial release, but preserved as candidate opportunities for V1.x or V2.0+.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    KINETIWALL NON-GOAL TAXONOMY                             │
├──────────────────────────────────────┬──────────────────────────────────────┤
│     PERMANENT PROJECT ANTI-GOALS     │     TEMPORARY V1.0 EXCLUSIONS        │
│   (Never Permitted in Any Release)   │(Excluded from V1.0; Roadmap Candidates│
├──────────────────────────────────────┼──────────────────────────────────────┤
│ • No Resident Gallery Background RAM │ • No Online Community Cloud Workshop │
│ • No Adware, Telemetry, or Spyware   │ • No In-App Visual Scene Graph Editor │
│ • No Mandatory Cloud Accounts / Subs │ • No Local LLM / AI Image Generation │
│ • No Windows Shell File Modification │ • No Dynamic Accent Color Sync       │
│ • No Audio Recording / Disk Storage  │ • No Direct Shadertoy URL Parsing    │
│ • No Unsandboxed Remote Executables  │ • No RGB Peripheral Hardware Sync    │
│ • No Unthrottled Busy-Wait Polling   │ • No Cross-Network Multi-PC Sync     │
│ • No Kernel-Mode Driver Injection    │ • No Linux (Wayland) / macOS Ports   │
└──────────────────────────────────────┴──────────────────────────────────────┘
```

---

## 2. Section 1: Permanent Project Anti-Goals

These 8 anti-goals represent foundational engineering contracts. They apply unconditionally to all versions of KinetiWall.

| Anti-Goal ID | Policy Statement | Rationale & User Impact | Enforcement Mechanism | Normative Authority | Approval Status |
|:---|:---|:---|:---|:---|:---:|
| **`KW-NG-001`** | **No Resident Gallery UI Background Process:** The gallery, settings, and property editor UI must completely terminate its process upon window close (`std::process::exit(0)`). It must never linger as a hidden, RAM-consuming background tray process. | Prevents the 70–120 MB RAM bloat common in Electron/Chromium wallpaper apps. Users expect an idle engine to consume only 8–12 MB. | Tauri v2 process termination on window close event. Zero resident WebView2 UI instances. | Foundational User Brief & Requirements Baseline | `APPROVED-DECISION` |
| **`KW-NG-002`** | **No Telemetry, Analytics, Tracking, or Spyware:** The application must never collect, aggregate, or transmit user behavioral data, wallpaper metadata, system specs, or desktop screenshots to any remote endpoint. | Protects user privacy, eliminates network overhead, and guarantees zero data harvesting. | Zero HTTP client libraries in daemon; automated network sniffer test suite in CI. | Foundational User Brief (`KW-NF-017`) | `APPROVED-DECISION` |
| **`KW-NG-003`** | **No Mandatory Cloud Accounts or Subscription Paywalls:** Core wallpaper playback, local importing, property customization, and multi-monitor management must operate 100% offline without requiring internet access or account creation. | The engine is a utility, not a SaaS product. Core functionality must never be gated behind an internet connection. | Air-gapped offline test suite (`CT-018`); zero cloud authentication dependencies. | Foundational User Brief (`KW-NF-012`) | `APPROVED-DECISION` |
| **`KW-NG-004`** | **No Windows Shell File Modification or Hook Injection:** The engine must never overwrite, patch, or modify protected Windows system files (`explorer.exe`, `uxtheme.dll`), nor inject DLLs into foreign processes. | Guarantees Windows stability, eliminates anti-cheat triggers (Vanguard/EasyAntiCheat), and ensures zero OS corruption. | Desktop attachment strictly uses public/undocumented window messaging (`0x052C`) without memory injection. | Systems Architecture Security Policy | `APPROVED-DECISION` |
| **`KW-NG-005`** | **No Audio Recording, Disk Serialization, or Streaming:** System audio captured via WASAPI loopback must exist exclusively in volatile memory buffers for transient FFT math. It must never be saved to disk, encoded to MP3/WAV, or streamed over a network. | Prevents security/privacy violations regarding desktop microphone or audio surveillance. | Volatile ring-buffer architecture with zero disk write routines in audio subsystem. | Step 1 Baseline (`KW-NF-014`) | `APPROVED-DECISION` |
| **`KW-NG-006`** | **No Unsandboxed Remote Executable Execution:** The engine must never download, unpack, and execute arbitrary unsigned native binaries (.exe, .dll, .bat) as live wallpapers without sandboxing and explicit user confirmation. | Prevents wallpaper packages from functioning as Trojan horses or malware delivery vectors. | Strict whitelist of supported media formats; native executables disabled by default. | Security Threat Model (`TR-006`) | `APPROVED-DECISION` |
| **`KW-NG-007`** | **No Unthrottled Busy-Wait Polling Loops:** Engine threads must never run unthrottled `while(true) { sleep(0); }` loops to check window occlusion, monitor status, or user input. | Eliminates unnecessary CPU power consumption and battery drain on mobile laptops. | 100% event-driven Win32 hooks (`SetWinEventHook`, `RegisterWindowMessageW`). | Performance Policy (`KW-NF-008`) | `APPROVED-DECISION` |
| **`KW-NG-008`** | **No Kernel-Mode Drivers or Custom Display Adapters:** The application must operate entirely in Windows User Mode (Ring 3). It must never install virtual display kernel drivers or filter drivers. | Eliminates Blue Screen of Death (BSOD) risks and driver signing certificate maintenance. | Standard Win32 User32 / D3D11 user-space graphics pipelines. | Platform Policy (`DEC-PLT-009`) | `APPROVED-DECISION` |

---

## 3. Section 2: Temporary Version 1.0 Release Exclusions

The following 10 capabilities are valuable enhancement candidates that are deliberately **excluded from Version 1.0** to ensure delivery velocity, architectural focus, and rock-solid reliability. They are cataloged in `docs/release_scope/OPTIONAL_AND_FUTURE_FEATURE_REGISTER.md` for post-V1 evaluation.

| Exclusion ID | Proposed Release Exclusion | Technical Rationale & Exclusion Basis | Related Candidate ID | Proposed Horizon | User Approval Impact |
|:---|:---|:---|:---:|:---:|:---:|
| **`V1-EXCL-001`** | **No In-App Visual Scene Graph Editor:** No node-based visual particle, shader, or logic editor within the Gallery UI in V1.0. | Building a visual scene editor requires massive front-end complexity (30k+ LOC) comparable to Wallpaper Engine's 8-year evolution. V1.0 focuses on running high-quality imported web/video content. | `OFF-009` / `GAP-02` | `V2.0+ CANDIDATE` | Preserves V1.0 launch schedule; wallpaper authors use standard web tools (VS Code, WebGL). |
| **`V1-EXCL-002`** | **No Centralized Cloud Workshop / Store:** No online wallpaper browsing, user accounts, ratings, or cloud sync in V1.0. | Cloud infrastructure introduces recurring server costs, content moderation liability, copyright abuse vectors, and violates offline-first focus. | `OFF-008` / `OPP-C01` | `V2.0+ CANDIDATE` | Preserves local-only zero-telemetry architecture for V1.0. |
| **`V1-EXCL-003`** | **No Bundled Local LLM / AI Image Generator:** No embedded ONNX/Stable Diffusion/Ollama models for AI wallpaper generation in V1.0. | Bundling AI weights inflates installer from 40 MB to 4+ GB, introduces extreme VRAM consumption (4–8 GB), and crashes lower-spec machines. | `OFF-011` / `OFF-012` | `EXPLORATION ONLY` | Prevents bloating installer and violating V1.0 RAM targets. |
| **`V1-EXCL-004`** | **No Dynamic OS Accent Color Harmonization:** No automated extraction of dominant wallpaper colors to update Windows 11 accent color palette. | Requires modifying Windows DWM/Shell registry keys and triggers visual flicker during rapid video playback. Best suited for V1.1 quality-of-life update. | `OFF-001` | `V1.x CANDIDATE` | Does not affect core wallpaper presentation. |
| **`V1-EXCL-005`** | **No Direct Shadertoy URL Import Engine:** No automated regex/GLSL translation layer converting raw Shadertoy scripts directly into wallpapers. | Shadertoy shaders utilize custom uniform variables (`iResolution`, `iChannel0..3`) and multi-pass buffers that require dedicated compilation transpilers. | `OFF-005` | `V1.x CANDIDATE` | WebGL wallpapers handle complex shaders via standard HTML packages in V1.0. |
| **`V1-EXCL-006`** | **No RGB Hardware Peripheral Synchronization:** No Razer Chroma, Corsair iCUE, or OpenRGB hardware lighting integration in V1.0. | Hardware SDKs introduce proprietary C++ DLL dependencies that frequently crash and conflict with user lighting software. | `OFF-010` | `V2.0+ CANDIDATE` | Keeps V1.0 engine lean and pure Rust. |
| **`V1-EXCL-007`** | **No Cross-Network Multi-PC Synchronization:** No LAN socket broadcasting to synchronize wallpaper playback across separate desktop PCs. | Niche use case with complex network discovery and clock synchronization challenges. | `FUTURE-OPP` | `V2.0+ CANDIDATE` | Multi-monitor on a single PC is 100% supported in V1.0. |
| **`V1-EXCL-008`** | **No Linux (Wayland) or macOS Operating System Ports:** No cross-platform desktop shells in V1.0. | Live wallpaper desktop attachment is completely OS-specific (Windows `WorkerW` vs Wayland `wlr-layer-shell` vs macOS `CGSSetWindowLevel`). Windows is the sole focus. | `OFF-009` / Step 4 Matrix | `V2.0+ CANDIDATE` | Windows desktop is the user's primary requested platform. |
| **`V1-EXCL-009`** | **No Standalone Game Engine (.exe) Window Reparenting:** Standalone compiled Godot/Unity Win32 executables are excluded from V1.0; 3D wallpapers use WebGL in WebView2. | Embedding external Win32 executables inside `WorkerW` causes input trapping, mouse stealing, and uncontrolled GPU crashes (`UDR-003`). | `KW-F-019b` / `VG-009` | `V2.0+ CANDIDATE` | WebGL 2.0 provides identical visual quality with 100% security and stability. |
| **`V1-EXCL-010`** | **No Microsoft Store MSIX Packaging:** V1.0 distributes exclusively via self-contained NSIS/WiX standalone installer. | MSIX container virtualization restricts access to `Progman` window handles and blocks clean Named Pipe IPC DACLs. | `DEC-PLT-008` | `V1.x CANDIDATE` | Standalone installer gives users 100% control and offline installation. |

---

## 4. Governance & Reclassification Rules

1. **Anti-Goals Are Permanent:** No feature request, customer feedback, or future roadmap proposal may overturn the 8 Permanent Project Anti-Goals without a formal architectural review and unanimous safety sign-off.
2. **Release Exclusion Re-evaluation:** Items classified under Section 2 (`V1-EXCL-001` through `010`) will be systematically re-evaluated at the conclusion of the Version 1.0 release cycle during Phase 6 release planning.
3. **Traceability:** Any attempt to introduce an excluded capability into V1.0 must be rejected by the technical program manager and logged in the project change register.

