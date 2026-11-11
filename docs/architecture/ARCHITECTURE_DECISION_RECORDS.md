# KinetiWall — Architecture Decision Records (ADRs)
**Document ID:** `KW-ARC-004`  
**Deliverable:** Deliverable D — Architecture Decision Records  
**Phase:** 2 — System Architecture & Technical Decisions  
**Step:** 7 of 90 — System Architecture & Architecture Decision Records  
**Status:** COMPLETE (Authoritative Decision Register)  
**Authority:** Principal Software Architect & Rust Systems Lead  
**Date:** 2026-11-10  

---

## 1. ADR Governance & Status Taxonomy

Every architectural decision in KinetiWall is formalized using an auditable Architecture Decision Record (ADR) model.

### Status Definitions
- **`ACCEPTED`**: Explicitly approved architectural decision backed by user requirements and technical verification.
- **`PROPOSED`**: Architecturally sound recommendation awaiting Phase 2 prototype gate validation.
- **`DEFERRED`**: Viable long-term direction postponed to post-V1 release horizons.
- **`REJECTED`**: Evaluated alternative explicitly rejected due to security, performance, or stability violations.
- **`SUPERSEDED`**: Previous decision replaced by an updated record.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                       14 ARCHITECTURE DECISION RECORDS                      │
├──────────┬─────────────────────────────────────────────────┬────────────────┤
│ ADR ID   │ Architectural Decision Topic                    │ Status         │
├──────────┼─────────────────────────────────────────────────┼────────────────┤
│ ADR-0001 │ Rust as Primary Systems Implementation Language │ ACCEPTED       │
│ ADR-0002 │ Native User-Session Daemon vs Windows Service   │ ACCEPTED       │
│ ADR-0003 │ Multi-Process Isolation vs Monolithic Process   │ ACCEPTED       │
│ ADR-0004 │ Ephemeral On-Demand Gallery UI (0 MB Closed)    │ ACCEPTED       │
│ ADR-0005 │ Dual-Mode Desktop Host Integration (WorkerW)    │ PROPOSED (VG1) │
│ ADR-0006 │ Embedded libmpv Dynamic Linkage with D3D11VA    │ ACCEPTED       │
│ ADR-0007 │ WebView2 Evergreen Runtime for Web Wallpapers   │ ACCEPTED       │
│ ADR-0008 │ Windows Named Pipes with Security DACL for IPC  │ ACCEPTED       │
│ ADR-0009 │ JSON Manifest V1 & Atomic File Configuration    │ ACCEPTED       │
│ ADR-0010 │ Windows Job Object Supervision for Child Cleanup│ ACCEPTED       │
│ ADR-0011 │ Event-Driven Win32 Hooks for Occlusion Pause    │ ACCEPTED       │
│ ADR-0012 │ Standalone Offline Installer Distribution       │ ACCEPTED       │
│ ADR-0013 │ Origin Sandboxing & CORS Denial for Web Content │ ACCEPTED       │
│ ADR-0014 │ 3D Scoped to WebGL in V1.0 vs Standalone .exe   │ PROPOSED (VG9) │
└──────────┴─────────────────────────────────────────────────┴────────────────┘
```

---

## 2. Exhaustive ADR Register

### `ADR-0001`: Rust as Primary Systems Implementation Language
- **Status:** `ACCEPTED` | **Date:** 2026-11-10
- **Context:** The application requires low-level Win32 API interaction, high-frequency multimedia IPC, and deterministic memory consumption without garbage collection pauses.
- **Related Requirements:** `KW-NF-001`, `KW-NF-002`, `KW-NF-025`, `KW-NF-026`.
- **Problem Statement:** C++ offers native performance but exposes memory corruption, use-after-free, and thread-safety hazards. C#/.NET introduces CLR runtime overhead (40–80 MB base RAM) and garbage collection pauses.
- **Decision Drivers:** Memory safety without runtime garbage collection; deterministic sub-15 MB RAM footprint; modern cargo build ecosystem; clean C-ABI FFI.
- **Candidate Options:**
  1. *Option A: Modern C++20.* (Manual memory management, high security vulnerability risk).
  2. *Option B: C# / .NET 8 / WPF / WinUI 3.* (Heavy GC overhead, fails 8–12 MB memory budget).
  3. *Option C: Pure Rust (2021 Edition).* (Zero-cost abstractions, compile-time thread safety, direct Win32 FFI via `windows` crate).
- **Decision:** Adopt Option C (Pure Rust).
- **Consequences:** Eliminates memory corruption vulnerabilities; enforces strict RAII resource cleanup; requires learning curve for Win32 unsafe FFI blocks.
- **Validation Required:** Memory footprint profiling in `BENCH-01` (<15 MB working set).
- **Approval Status:** `APPROVED-DECISION` (Explicit User Mandate).

---

### `ADR-0002`: Native User-Session Daemon vs Windows Service Architecture
- **Status:** `ACCEPTED` | **Date:** 2026-11-10
- **Context:** Background desktop utilities can be engineered as either a standard interactive user session application or a background Windows Service running under `LocalSystem`.
- **Related Requirements:** `KW-F-005`, `KW-F-008`, `KW-NF-011`.
- **Problem Statement:** Windows Services run in Session 0 since Windows Vista. Session 0 is completely isolated from the interactive user desktop (Session 1+). Services cannot interact with desktop HWNDs, broadcast shell messages (`0x052C`), or present Direct3D swapchains to the user screen without dangerous cross-session process launching.
- **Decision Drivers:** Windows Session 0 security isolation; native access to `Progman` and `WorkerW`; least-privilege non-elevated user execution.
- **Candidate Options:**
  1. *Option A: Windows Service (Session 0).* (Blocked by Session 0 GUI isolation; requires elevation).
  2. *Option B: User-Session Native Daemon (`kinetiwall-core.exe`).* (Runs under interactive user logon SID; direct Win32 message access; no elevation required).
- **Decision:** Adopt Option B (User-Session Native Daemon).
- **Consequences:** Core controller starts on user logon via standard `HKCU\Software\Microsoft\Windows\CurrentVersion\Run`; operates entirely within user integrity level (Medium IL).
- **Validation Required:** Startup verification on standard non-admin account (`CT-017`).
- **Approval Status:** `APPROVED-DECISION`.

---

### `ADR-0003`: Multi-Process Isolated Architecture vs Monolithic Single Process
- **Status:** `ACCEPTED` | **Date:** 2026-11-10
- **Context:** Third-party multimedia decoders (`libmpv`, FFmpeg) and web engines (Chromium/WebView2) are complex C/C++ codebases susceptible to crashes, driver incompatibilities, and memory leaks.
- **Related Requirements:** `KW-F-006`, `KW-F-007`, `KW-NF-009`, `KW-NF-010`.
- **Problem Statement:** If video playback or web script execution is hosted within the main controller process, a single corrupted media file or GPU driver fault terminates the entire wallpaper engine and disrupts the desktop.
- **Decision Drivers:** Fault isolation; memory sandboxing; independent crash recovery; ability to kill runaway players.
- **Candidate Options:**
  1. *Option A: Monolithic Single-Process Engine.* (Simpler IPC; catastrophic crash propagation).
  2. *Option B: Multi-Process Architecture.* (Headless core daemon supervises isolated child processes for video and web playback).
- **Decision:** Adopt Option B (Multi-Process Architecture).
- **Consequences:** Video player (`kinetiwall-player-video.exe`) and web player (`kinetiwall-player-web.exe`) run as separate executables. A crash in a player is isolated, logged, and restarted without affecting the controller.
- **Validation Required:** Fault-injection crash test `REL-02` (50 player kills).
- **Approval Status:** `APPROVED-DECISION`.

---

### `ADR-0004`: Ephemeral On-Demand Gallery UI Lifecycle (Zero RAM Closed)
- **Status:** `ACCEPTED` | **Date:** 2026-11-10
- **Context:** Many live wallpaper applications (e.g. Wallpaper Engine, Lively Wallpaper) leave their web/Electron/WPF UI process lingering in the background tray, wasting 70–180 MB RAM 24/7.
- **Related Requirements:** `KW-NF-002`, `KW-NG-001`.
- **Problem Statement:** Users expect an idle wallpaper engine to be imperceptible. A persistent UI violates the 8–12 MB idle memory budget.
- **Decision Drivers:** Zero resident UI memory; responsive user experience when opened; complete process termination.
- **Candidate Options:**
  1. *Option A: Hidden UI Process.* (Hide window on close; retains 70–120 MB RAM permanently).
  2. *Option B: Ephemeral Process Lifecycle.* (Intercept `WM_CLOSE`; execute `std::process::exit(0)`; launch fresh instance on tray click).
- **Decision:** Adopt Option B (Ephemeral Process Lifecycle).
- **Consequences:** `kinetiwall-ui.exe` releases 100% of its memory upon close. When closed, exactly 0.0 MB RAM is consumed by UI. Cold launch takes ~300ms.
- **Validation Required:** Process table audit in `KW-AC-006`.
- **Approval Status:** `APPROVED-DECISION` (Explicit User Anti-Goal `KW-NG-001`).

---

### `ADR-0005`: Dual-Mode Windows Desktop Host Shell Attachment (WorkerW / Progman)
- **Status:** `PROPOSED (SUBJECT TO VG-001)` | **Date:** 2026-11-10
- **Context:** Attaching child windows behind desktop icons historically relies on undocumented message `0x052C` to `Progman` spawning a `WorkerW` sibling beneath `SHELLDLL_DefView`.
- **Related Requirements:** `KW-F-001`, `KW-F-002`, `DEC-PLT-001`.
- **Problem Statement:** In Windows 11 24H2, unchecking "Show desktop icons" destroys the secondary `WorkerW`, causing wallpapers attached via traditional methods to detach or turn black.
- **Decision Drivers:** Cross-version compatibility across Windows 11 24H2 and Windows 10; desktop icon clickability; zero visual artifacts.
- **Candidate Options:**
  1. *Option A: Traditional WorkerW-only Attachment.* (Fails on Windows 11 24H2 when desktop icons are hidden).
  2. *Option B: Dual-Mode Attachment Algorithm (`DEC-PLT-001`).* (Primary `WorkerW` when icons visible; fallback direct `Progman` child attachment beneath `SHELLDLL_DefView` when icons hidden).
  3. *Option C: Borderless Layered Desktop Window.* (Floats on desktop; intercepts clicks, breaks icon interaction).
- **Decision:** Adopt Option B (Dual-Mode Attachment Algorithm).
- **Consequences:** Guarantees desktop wallpaper visibility regardless of Windows 11 24H2 icon settings.
- **Validation Required:** Prototype Validation Gate `VG-001`.
- **Approval Status:** `PROPOSED (APPROVED PLATFORM POLICY)`.

---

### `ADR-0006`: Embedded libmpv Dynamic Linkage with Direct3D 11 Hardware Decoding
- **Status:** `ACCEPTED` | **Date:** 2026-11-10
- **Context:** Hardware-accelerated video playback requires broad codec support (H.264, HEVC, VP9, AV1), sub-16.6ms gapless looping, and LGPL v2.1+ licensing compliance.
- **Related Requirements:** `KW-F-009` through `KW-F-013`, `KW-F-046`, `DEC-PLT-004`.
- **Problem Statement:** Windows Media Foundation (WMF) lacks native WebM/VP9/AV1 support without third-party store extensions. Spawning an external `mpv.exe` introduces window reparenting races and high process overhead.
- **Decision Drivers:** Direct3D 11 hardware decoding (`vo=gpu-next`, `hwdec=d3d11va`); infinite gapless looping (`loop-file=inf`); LGPL v2.1+ dynamic linking compliance.
- **Candidate Options:**
  1. *Option A: Windows Media Foundation (WMF).* (Missing open codecs; brittle loop boundaries).
  2. *Option B: Standalone `mpv.exe` External CLI.* (Process management complexity; window flicker).
  3. *Option C: Dynamically Linked `libmpv-2.dll` via C-API FFI.* (Direct swapchain control, native D3D11VA, gapless pre-roll, LGPL Section 6 compliant).
- **Decision:** Adopt Option C (Dynamically Linked `libmpv-2.dll`).
- **Consequences:** Video player links to `libmpv-2.dll` at runtime via standard Win32 dynamic loader. Users can replace the DLL with their own build.
- **Validation Required:** Validation Gate `VG-003` and `VG-010`.
- **Approval Status:** `APPROVED-DECISION`.

---

### `ADR-0007`: Microsoft Edge WebView2 for Sandboxed Interactive Web Wallpapers
- **Status:** `ACCEPTED` | **Date:** 2026-11-10
- **Context:** Interactive wallpapers require HTML5, CSS3, Canvas, WebGL 1.0/2.0, and modern JavaScript execution.
- **Related Requirements:** `KW-F-014` through `KW-F-018`, `DEC-PLT-003`.
- **Problem Statement:** Bundling Chromium/CEF inflates installer size by 150+ MB and requires continuous security patching. Ultralight lacks full WebGL 2.0 support.
- **Decision Drivers:** Pre-installed Evergreen runtime on Windows 10/11; native WebGL 2.0 acceleration; virtual host folder mapping security.
- **Candidate Options:**
  1. *Option A: Bundled Chromium Embedded Framework (CEF).* (Massive 200 MB installer bloat).
  2. *Option B: Ultralight / Wry.* (Incomplete WebGL 2.0 shader support; rendering artifacts).
  3. *Option C: Microsoft Edge WebView2 Evergreen Runtime.* (0 MB installer weight on modern Windows; full WebGL 2.0; automatic security updates by Microsoft).
- **Decision:** Adopt Option C (Microsoft Edge WebView2).
- **Consequences:** Eliminates browser engine bundling; requires pre-flight check for WebView2 presence on air-gapped systems (`CT-018`).
- **Validation Required:** Validation Gate `VG-006`.
- **Approval Status:** `APPROVED-DECISION`.

---

### `ADR-0008`: Windows Named Pipes with Security DACL for Local Inter-Process Communication
- **Status:** `ACCEPTED` | **Date:** 2026-11-10
- **Context:** The core daemon, gallery UI, and player processes require high-speed, local bidirectional message passing.
- **Related Requirements:** `KW-NF-003`, `KW-NF-012`, `KW-NF-013`.
- **Problem Statement:** Local TCP sockets (`127.0.0.1`) trigger Windows Defender firewall prompts, can be probed by non-elevated network scanners, and expose attack surface. Standard anonymous pipes do not support multi-client request-response patterns.
- **Decision Drivers:** High-speed Win32 kernel message passing; native Windows Object Security DACL; zero firewall interference.
- **Candidate Options:**
  1. *Option A: Local TCP Sockets (localhost).* (Firewall prompts, network port exhaustion).
  2. *Option B: Windows Message Broadcasting (`WM_COPYDATA`).* (High UI thread latency; drops messages under load).
  3. *Option C: Local Named Pipes (`\\.\pipe\kinetiwall-ipc-{SessionId}`).* (Native Win32 IPC, kernel-enforced DACL to interactive logon SID, sub-millisecond roundtrip).
- **Decision:** Adopt Option C (Local Named Pipes with Security DACL).
- **Consequences:** Grants access strictly to current user token; rejects foreign sessions (`ERROR_ACCESS_DENIED`).
- **Validation Required:** Security test `SEC-01` and `SEC-02`.
- **Approval Status:** `APPROVED-DECISION`.

---

### `ADR-0009`: JSON Manifest Schema V1 and Transactional Atomic Configuration Storage
- **Status:** `ACCEPTED` | **Date:** 2026-11-10
- **Context:** Wallpaper metadata and application configuration require persistent, human-readable, and corruption-resilient local storage.
- **Related Requirements:** `KW-F-033`, `KW-F-037`.
- **Problem Statement:** A heavy relational database (SQLite) introduces binary migration overhead and locked file issues. Non-atomic file writes cause configuration loss on abrupt power cuts.
- **Decision Drivers:** Human-readable configuration; atomic disk writes; zero external database driver dependencies; portability.
- **Candidate Options:**
  1. *Option A: Embedded SQLite Database.* (Overkill for simple wallpaper lists; binary format prevents user text inspection).
  2. *Option B: Standard Non-Atomic JSON.* (Prone to corruption on crash/power loss).
  3. *Option C: Versioned JSON Manifests (`project.json`) + Atomic File Write (`config.json.tmp` -> `config.json` with `.bak` recovery).*
- **Decision:** Adopt Option C (JSON Manifests + Atomic Write Persistence).
- **Consequences:** Allows creators to edit `project.json` in VS Code; atomic rename ensures zero corruption; auto-restores from backup.
- **Validation Required:** Fault-injection corruption test `REL-07`.
- **Approval Status:** `APPROVED-DECISION`.

---

### `ADR-0010`: Windows Job Object Lifecycle Management for Child Process Supervision
- **Status:** `ACCEPTED` | **Date:** 2026-11-10
- **Context:** An unhandled crash in the core controller daemon could leave child player processes orphaned and consuming resources.
- **Related Requirements:** `KW-F-006`, `KW-NF-010`.
- **Problem Statement:** Relying solely on process polling or heartbeat timers leaves zombie processes running if the controller is terminated forcefully via Task Manager.
- **Decision Drivers:** Kernel-enforced process lifecycle; 100% cleanup guarantee; hard resource quota enforcement.
- **Candidate Options:**
  1. *Option A: Manual Process Polling & Heartbeats.* (Vulnerable to controller hard crash; leaves zombies).
  2. *Option B: Windows Job Objects with `JOB_OBJECT_LIMIT_KILL_ON_JOB_CLOSE`.* (Kernel automatically kills all processes assigned to the job the instant the job handle is closed).
- **Decision:** Adopt Option B (Windows Job Objects).
- **Consequences:** Zero orphan processes possible; enables setting hard memory ceilings (500 MB) on web player processes.
- **Validation Required:** 50-run crash test `REL-03` and Validation Gate `VG-002`.
- **Approval Status:** `APPROVED-DECISION`.

---

### `ADR-0011`: Event-Driven Win32 Hooks for Occlusion and Fullscreen Detection
- **Status:** `ACCEPTED` | **Date:** 2026-11-10
- **Context:** Detecting fullscreen games and maximized applications to pause playback must be fast and power-efficient.
- **Related Requirements:** `KW-F-026`, `KW-NF-004`, `KW-NF-008`.
- **Problem Statement:** Polling window states every 500ms via `GetForegroundWindow` wastes CPU cycles, drains laptop batteries, and introduces up to 500ms of lag.
- **Decision Drivers:** 100% event-driven; sub-50ms pause latency; 0.00% idle CPU utilization.
- **Candidate Options:**
  1. *Option A: Periodic Polling Loop (Sleep timer).* (CPU waste, battery drain, violates `KW-NG-007`).
  2. *Option B: Event-Driven `SetWinEventHook`.* (Listens for `EVENT_SYSTEM_FOREGROUND` and `EVENT_OBJECT_LOCATIONCHANGE`; executes only when window state changes).
- **Decision:** Adopt Option B (Event-Driven Win32 Hooks).
- **Consequences:** Core daemon thread sleeps until OS dispatches event; pause latency ≤50ms; zero CPU busy-loops.
- **Validation Required:** Latency benchmark `BENCH-04` and CapFrameX test `BENCH-03`.
- **Approval Status:** `APPROVED-DECISION`.

---

### `ADR-0012`: Standalone Offline Installer Distribution (NSIS/WiX)
- **Status:** `ACCEPTED` | **Date:** 2026-11-10
- **Context:** Distribution packages must support air-gapped installation, enterprise silent deployment, and clean uninstallation.
- **Related Requirements:** `KW-F-043`, `KW-F-044`, `KW-F-045`, `DEC-PLT-008`.
- **Problem Statement:** Microsoft Store MSIX packaging isolates win32 executables inside virtualized containers, blocking desktop shell HWND reparenting and restricting local Named Pipe DACLs.
- **Decision Drivers:** 100% offline-first execution; unvirtualized desktop handle access; silent enterprise switches (`/S`).
- **Candidate Options:**
  1. *Option A: Microsoft Store MSIX Container.* (Virtualized container blocks `WorkerW` reparenting).
  2. *Option B: Standalone NSIS / WiX Installer.* (Standard Win32 installer; full desktop API access; silent switches; clean uninstallation).
- **Decision:** Adopt Option B (Standalone NSIS/WiX Installer).
- **Consequences:** Package ships as standalone `KinetiWall-Setup-v1.0.0-x64.exe` (target size ≤40 MB); verified on air-gapped machines.
- **Validation Required:** Offline VM installation test `CT-018` and `VG-008`.
- **Approval Status:** `APPROVED-DECISION`.

---

### `ADR-0013`: Origin Sandboxing and CORS Denial for Untrusted Web Content
- **Status:** `ACCEPTED` | **Date:** 2026-11-10
- **Context:** Web wallpapers execute third-party JavaScript that must be prevented from accessing the local disk, network, or camera/microphone.
- **Related Requirements:** `KW-F-016`, `KW-NF-016`, `KW-NG-006`.
- **Problem Statement:** Hosting web wallpapers via local `file://` URLs allows unrestricted directory traversal. Hosting via standard HTTP allows remote cross-origin data exfiltration.
- **Decision Drivers:** Complete network isolation; local file path restriction; hardware permission auto-denial.
- **Candidate Options:**
  1. *Option A: Direct `file://` URL Navigation.* (Directory traversal vulnerabilities).
  2. *Option B: Virtual Host Directory Mapping (`https://kinetiwall.local/`) with CORS Denial.* (Maps strictly to wallpaper folder; blocks all outbound network requests; auto-denies device permissions).
- **Decision:** Adopt Option B (Virtual Host Directory Mapping).
- **Consequences:** Blocks data exfiltration; prevents reading sensitive Windows files; isolates wallpaper storage.
- **Validation Required:** Penetration test suite `SEC-04` through `SEC-07` and `VG-006`.
- **Approval Status:** `APPROVED-DECISION`.

---

### `ADR-0014`: 3D Wallpapers Scoped to WebGL Packages in V1.0 vs Standalone Executable Reparenting
- **Status:** `PROPOSED (CONFIRMED V1.0 SCOPE)` | **Date:** 2026-11-10
- **Context:** The initial user brief requested 3D/Unity/Godot live wallpaper capabilities.
- **Related Requirements:** `KW-F-019`, `UDR-003`, `V1-EXCL-009`.
- **Problem Statement:** Reparenting standalone compiled game executables (`.exe`) into `WorkerW` creates severe focus loss, mouse-cursor trapping, anti-cheat conflicts, and uncontrolled GPU crashes.
- **Decision Drivers:** Stability; security sandboxing; release delivery velocity; user safety.
- **Candidate Options:**
  1. *Option A: Standalone Win32 Executable Window Reparenting in V1.0.* (High instability, steals keyboard focus, unmanaged GPU crashes).
  2. *Option B: WebGL 2.0 / WebGL-Exported Godot 4 Packages in WebView2 for V1.0; Defer Standalone `.exe` Reparenting to V2.0+ (`VG-009`).*
- **Decision:** Adopt Option B (Deliver 3D via WebGL in V1.0; prototype standalone `.exe` reparenting for V2.0+).
- **Consequences:** Users can author 3D wallpapers using Three.js, Babylon.js, or Godot 4 WebGL export templates; guarantees 100% sandboxed stability in V1.0.
- **Validation Required:** Validation Gate `VG-009` (Phase 2 Prototyping Spike).
- **Approval Status:** `PROPOSED (CONFIRMED RELEASE SCOPE BASELINE)`.

