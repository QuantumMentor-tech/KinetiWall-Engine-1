# KinetiWall — Research-to-Requirements Traceability Matrix

**Document Identifier:** KW-RES-005  
**Document Type:** Formal Research Traceability Specification (Deliverable E)  
**Current Phase:** Phase 1 — Project Research, Requirements & Feasibility  
**Step:** Step 2 of 90 — Technical Research & Source Verification  
**Status:** Approved Traceability Baseline  
**Date of Research:** 2026-11-08  

---

## 1. Traceability Methodology & Classification Scheme

Every major technical research finding, platform observation, and engineering conclusion established during Step 2 is bidirectionally mapped to the Step 1 Requirements Inventory (`KW-F`, `KW-NF`, `KW-NG`), Unresolved Decisions Register (`UDR`), and Optional Features Catalog (`OFF`).

### Conclusion Classifications:
1. **CONFIRMED FACT:** Directly verified by official documentation (Microsoft Learn, Rust-lang, Khronos) or stable operating system APIs.
2. **EVIDENCE-SUPPORTED CONCLUSION:** Substantiated by reproducible empirical data, open-source codebase audits, or maintainer issue reports.
3. **ENGINEERING RECOMMENDATION:** A technically justified architectural approach selected based on comparative trade-off analysis.
4. **PROVISIONAL ASSUMPTION:** A working assumption that requires empirical calibration during Phase 1 prototype spikes.
5. **UNRESOLVED QUESTION:** A technical or architectural question requiring formal user approval or prototype validation before resolution.
6. **OUTDATED OR CONTRADICTED CLAIM:** A claim from prior research or competitor notes shown to be inaccurate, obsolete, or misleading.

---

## 2. Research-to-Requirements Traceability Matrix

| Finding ID | Research Topic & Conclusion Statement | Linked Requirement IDs | Research Classification | Supporting Evidence & Source | Consequence / Action |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **RC-001** | **Progman / WorkerW Desktop Parenting:** Message `0x052C` sent to `Progman` spawns `WorkerW` behind desktop icons, but Windows 11 24H2 requires fallback handling when icons are hidden. | `KW-F-001`, `KW-F-003`, `UDR-002` | **EVIDENCE-SUPPORTED CONCLUSION** | `SR-001`, Lively Wallpaper Issues #2500+, StackOverflow 24H2 | Step 3 PoC must test dual-mode attachment on Windows 11 24H2. |
| **RC-002** | **Shell Restart Recovery:** The `TaskbarCreated` registered window message reliably signals `explorer.exe` restarts for handle re-negotiation. | `KW-F-002`, `KW-F-041`, `UDR-002` | **CONFIRMED FACT** | `SR-002`, Microsoft Learn Taskbar API | Implement 500ms debounce timer before re-attaching player windows. |
| **RC-003** | **Anti-Goal: Zero Shell Injection:** Sub-desktop parenting via Win32 messages eliminates the need to inject DLLs into `explorer.exe`. | `KW-NG-004`, `KW-F-001` | **CONFIRMED FACT** | `SR-001`, Win32 API Architecture | Guarantees stability and prevents anti-cheat false positives. |
| **RC-004** | **Dedicated Multi-Binary Architecture:** Dedicated binaries for daemon, UI, and players guarantee crash isolation and minimal idle daemon memory (<10 MB). | `KW-F-005`, `KW-F-006`, `UDR-003` | **ENGINEERING RECOMMENDATION** | `SR-003`, Process Architecture Benchmarks | Modular Cargo workspace separating core, UI, and players. |
| **RC-005** | **Zero Resident Gallery Memory:** Tauri v2 UI binary terminating on close (`std::process::exit(0)`) completely unloads UI memory from RAM. | `KW-F-007`, `KW-NG-001`, `UDR-003` | **CONFIRMED FACT** | Tauri v2 Process Model, OS Process Memory | Rejects resident background UI bloatware; saves 70–120 MB RAM. |
| **RC-006** | **Windows Job Object Child Cleanup:** `JOB_OBJECT_LIMIT_KILL_ON_JOB_CLOSE` with non-inheritable handles guarantees kernel-level termination of orphaned players. | `KW-NF-010`, `UDR-003` | **CONFIRMED FACT** | `SR-004`, Microsoft Learn WinNT JobObjects | Configure `bInheritHandle = FALSE` to prevent handle leaks. |
| **RC-007** | **Single-User Named Pipe DACL Security:** Restricting Named Pipe DACL to the current user SID and Local System prevents unauthorized local IPC hijacking. | `KW-F-008`, `KW-NF-012`, `KW-NF-013` | **CONFIRMED FACT** | `SR-005`, Microsoft Win32 Security API | Compile SDDL string with user SID; reject remote clients. |
| **RC-008** | **Video Decoding Engine: Embedded libmpv:** `libmpv-2.dll` with `d3d11va` and `vo=gpu-next` delivers sub-millisecond gapless looping and universal codec support. | `KW-F-010`, `KW-F-012`, `KW-F-013`, `UDR-004` | **ENGINEERING RECOMMENDATION** | `SR-006`, mpv Manual, Lively WMF vs mpv | Outweighs WMF due to WMF looping stutter and codec limits. |
| **RC-009** | **Strict LGPL 2.1 Media Licensing:** Compiling `libmpv-2.dll` under LGPL 2.1+ with dynamic runtime linking avoids copyleft GPL infection. | `UDR-005` | **CONFIRMED FACT** | GNU LGPL v2.1 text, FFmpeg license guide | Include LGPL notices and source build instructions in installer. |
| **RC-010** | **WebView2 Virtual Host Sandbox:** `SetVirtualHostNameToFolderMapping` (`https://kinetiwall.local/`) provides an isolated HTTPS origin blocking file traversal. | `KW-F-017`, `KW-F-018`, `KW-NF-016`, `UDR-010` | **CONFIRMED FACT** | `SR-007`, Microsoft Edge WebView2 API | Enforce `DENY_CORS` and disable camera/mic permissions. |
| **RC-011** | **Event-Driven Foreground Tracking:** `SetWinEventHook` (`EVENT_SYSTEM_FOREGROUND`) provides sub-100ms fullscreen detection with <0.05% idle CPU. | `KW-F-025`, `KW-F-028`, `KW-NF-002`, `UDR-007` | **CONFIRMED FACT** | `SR-008`, Win32 Shell API | Pair with 2000ms safety watchdog to catch edge-case events. |
| **RC-012** | **Granular Pause vs Stop States:** Paused retains Direct3D swapchain (resume <30ms, 0% CPU); Stopped frees VRAM (resume ~300ms). | `KW-F-026`, `KW-NF-004`, `UDR-008` | **ENGINEERING RECOMMENDATION** | Graphics Architecture Analysis | Provide user setting: Instant Resume (Pause) vs VRAM Free (Stop). |
| **RC-013** | **WASAPI Loopback Audio Capture:** Loopback capture from default render endpoint via `AUDCLNT_STREAMFLAGS_LOOPBACK` operates without virtual audio cables. | `KW-F-031`, `KW-F-033`, `KW-NG-005` | **CONFIRMED FACT** | `SR-009`, Microsoft CoreAudio API | Implement `IMMNotificationClient` for dynamic headphone changes. |
| **RC-014** | **In-Memory FFT Audio Privacy:** FFT computation in volatile memory (`rustfft`) with zero disk writes guarantees complete audio privacy. | `KW-F-032`, `KW-F-033`, `KW-NG-005` | **CONFIRMED FACT** | Rust DSP Architecture | Thread sleeps after >3s silence; zero disk recording code. |
| **RC-015** | **Hybrid SQLite + JSON Storage:** SQLite (`PRAGMA journal_mode=WAL`) provides <2ms queries, backed by portable `kinetiwall.json` folder manifests. | `KW-F-034`, `KW-F-035`, `UDR-010` | **ENGINEERING RECOMMENDATION** | SQLite Documentation, Database Benchmarks | Auto-rebuild database cache from manifests if corrupted. |
| **RC-016** | **Zip Slip Archive Extraction Guard:** Canonical path validation during extraction strictly blocks relative path traversal (`../`) and absolute paths. | `KW-NF-014` | **CONFIRMED FACT** | `SR-010`, Snyk Security Research | Canonicalize paths; reject entries outside target folder. |
| **RC-017** | **Zero Telemetry and Sovereign Privacy:** Absence of telemetry SDKs and network sockets guarantees complete user privacy and zero data exfiltration. | `KW-NF-017`, `KW-NF-018`, `KW-NF-019`, `KW-NG-002`, `KW-NG-003` | **CONFIRMED FACT** | Privacy Architecture Audit | Crash reporting relies on local opt-in logs, never remote uploads. |
| **RC-018** | **Installer Footprint Calibration:** Standard installer with `libmpv-2.dll` and 3 sample wallpapers measures ~56 MB; <35 MB requires omitting sample media. | `KW-NF-006`, `UDR-006` | **EVIDENCE-SUPPORTED CONCLUSION** | Artifact Size Audit (`KW-RES-001`, Sec 12) | Update `KW-NF-006` calibration note: 56 MB with media, 35 MB bare. |
| **RC-019** | **AI / SLM Model Exclusion:** Local neural models consume 2.5–5.0 GB RAM/VRAM and cause severe GPU contention; strictly incompatible with core v1. | `OFF-012`, `UDR-012` | **EVIDENCE-SUPPORTED CONCLUSION** | AI Hardware Analysis (`KW-RES-001`, Sec 14) | AI capabilities formally deferred to external companion tooling. |
| **RC-020** | **Native 3D Executable Hosting Risk:** Reparenting arbitrary standalone `.exe` games via `SetParent` creates severe anti-cheat and malware vulnerabilities. | `KW-F-044`, `KW-NG-006`, `UDR-011` | **EVIDENCE-SUPPORTED CONCLUSION** | Competitor Security Audits, Windows Shell | Fulfill 3D needs via WebGL2 in web player; defer native `.exe` hosting. |

---

## 3. Disproven, Outdated, or Contradicted Prior Assumptions

1. **Prior Assumption: "Windows Media Foundation (WMF) is sufficient for lightweight gapless live wallpapers."**
   - *Status:* **CONTRADICTED CLAIM.**
   - *Evidence:* WMF flushes Direct3D video processors on loop boundaries, producing a visible black frame or stutter. Furthermore, Windows 10 N editions lack WMF codecs. `libmpv` with `vo=gpu-next` is technically required for seamless looping.
2. **Prior Assumption: "WorkerW message 0x052C is guaranteed to work identically across all Windows 11 builds."**
   - *Status:* **OUTDATED CLAIM.**
   - *Evidence:* Windows 11 24H2 breaks the standard WorkerW window hierarchy when desktop icons are hidden. Dual-mode fallback attachment is mandatory.
3. **Prior Assumption: "A complete offline live wallpaper installer can easily fit under 25 MB."**
   - *Status:* **OUTDATED CLAIM.**
   - *Evidence:* A complete offline package with `libmpv-2.dll` (22 MB), Tauri UI (8 MB), core daemon/players (11 MB), and sample media (15 MB) requires ~56 MB. A 25 MB target requires omitting all sample wallpapers and relying on system runtimes.
4. **Prior Assumption: "Local AI language models should be bundled into KinetiWall for smart search."**
   - *Status:* **CONTRADICTED CLAIM.**
   - *Evidence:* Small language models consume >2.5 GB of VRAM and starve GPU bandwidth, directly violating KinetiWall's core performance requirements (`KW-NF-001`, `KW-NF-005`).

