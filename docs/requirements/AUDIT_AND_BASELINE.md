# KinetiWall — Repository Audit & Requirements Baseline

**Document Identifier:** KW-DOC-001  
**Document Type:** Engineering Baseline, Audit Report & Governance Specification  
**Current Phase:** Phase 1 — Project Research, Requirements & Feasibility  
**Step:** Step 1 of 90 — Project Requirements Inventory & Traceability  
**Status:** Approved Requirements Baseline  
**Date:** 2026-11-08  

---

## 1. Executive Summary

This document establishes the official engineering audit and requirements baseline for **KinetiWall**, a high-performance Windows desktop customization and live wallpaper engine engineered primarily in **Rust**. KinetiWall delivers seamless, hardware-accelerated video, dynamic web/shader, and interactive wallpapers beneath Windows desktop icons while enforcing strict resource efficiency, battery-conscious power management, multi-monitor flexibility, robust crash isolation, and local-first privacy.

This document records the pre-implementation state of the repository, defines the source precedence and authority hierarchy, formalizes the core product identity, and details the governance rules ensuring traceable implementation throughout the 90-step development lifecycle.

---

## 2. Pre-Implementation Workspace & Git Audit

In compliance with Step 1 audit protocols, a comprehensive pre-implementation inspection of the local workspace environment was conducted prior to modifying or creating any documentation.

### 2.1 Workspace Inspection
- **Project Root Directory:** `E:\Projects\All working projects\App projects working\KinetiWall Engine (working)`
- **Initial Workspace Contents:**
  - `KinetiWall Neon Glass Logo.png` (Size: 538,802 bytes, Timestamp: 2026-11-08 10:53 AM).
  - No application source code (`src/`, `Cargo.toml`, `package.json`, or frontend directories) was present.
  - No pre-existing requirements, architecture, or planning documentation was present.
- **Repository Classification:** Clean greenfield workspace initialized with brand visual assets.

### 2.2 Version Control (Git) Safety Inspection
- **Git Initialization Status:** Git is currently **not initialized** (`fatal: not a git repository`).
- **Working Tree State:** Greenfield directory containing one pre-existing asset (`KinetiWall Neon Glass Logo.png`).
- **Preservation Verification:** The pre-existing logo file remains completely untouched. No files were deleted, moved, or overwritten.
- **Git Action Policy for Step 1:** In accordance with Section 4.2 and Section 15 of the Step 1 specification, no artificial git commits or destructive repository operations were executed. Version control initialization and initial commit policies are preserved for subsequent workspace setup steps as authorized by the user.

### 2.3 Toolchain and Runtime Environment Inspection
The local Windows host environment was inspected for relevant development runtimes without installing or modifying dependencies:
- **Rust Compiler (`rustc`):** Version `1.99.0` (commit `b940084d7 2026-09-28`) — Installed and active.
- **Cargo Package Manager (`cargo`):** Version `1.99.0` (commit `5f94df478 2026-08-27`) — Installed and active.
- **Node.js Runtime (`node`):** Version `v26.8.1` — Installed and active.
- **Node Package Manager (`npm`):** Version `11.19.0` — Installed and active.
- **Host Operating System:** Windows 10/11 Desktop 64-bit (`x86_64-pc-windows-msvc`).
- **Toolchain Status:** The development host possesses modern, production-grade Rust and Node.js environments capable of supporting the proposed multi-process Rust architecture and potential web-based presentation tooling.

---

## 3. Authoritative Inputs & Source Precedence

To maintain absolute requirements integrity and prevent scope creep, every requirement, design constraint, and candidate feature is governed by a strict four-tier precedence hierarchy.

```
+-------------------------------------------------------------+
| Tier 1: Explicit User Requirements & Directives (Authoritative)  |
| - Product Identity: KinetiWall                              |
| - Platform: Windows Desktop & Laptops                       |
| - Primary Language: Rust                                    |
| - Core Scope: Video, Web, Audio-Reactive, Multi-Monitor     |
| - Explicit Non-Goals: No resident gallery UI, no spyware    |
+-------------------------------------------------------------+
                              |
                              v
+-------------------------------------------------------------+
| Tier 2: Approved Project Decisions (Binding Architectural Baseline) |
| - Multi-process architecture (Core controller + Players)   |
| - Offline-first operation & zero required accounts          |
| - Formats requested: MP4, WebM, AVI                         |
+-------------------------------------------------------------+
                              |
                              v
+-------------------------------------------------------------+
| Tier 3: Gemini Research Document & Competitor Analysis      |
| - Candidate technologies: libmpv vs Media Foundation        |
| - Undocumented Progman/WorkerW 0x052C attachment            |
| - WebView2 integration architecture                         |
| - MKV container support & extended codecs                   |
| - Provisional performance targets (<10MB RAM, <0.1% CPU)    |
+-------------------------------------------------------------+
                              |
                              v
+-------------------------------------------------------------+
| Tier 4: Optional, Future, & Unresolved Decisions            |
| - Standalone 3D / Game engine integration                   |
| - Accent color sync, Shadertoy import, HUD switcher         |
| - Local AI frame interpolation / generation                 |
+-------------------------------------------------------------+
```

### 3.1 Tier 1: Explicit User Requirements (Authoritative)
The user's direct instructions establish:
- **Product Name:** KinetiWall.
- **Core Functionality:** Windows live wallpaper application rendering behind desktop icons with video playback, interactive web wallpapers, audio visualization, multi-monitor management, and resource-aware pausing.
- **Programming Language:** Rust is the explicit primary language preference.
- **Operating Model:** Local-first, privacy-respecting, and fully operational offline.
- **Critical Non-Goals:** The gallery management UI must **not** persist as a hidden, resident background process when closed; no bundled telemetry spyware or data collection.

### 3.2 Tier 2: Approved Project Decisions (Binding Architectural Baseline)
Architectural decisions explicitly approved during project scoping:
- Multi-process separation: isolated wallpaper player processes supervised by a lightweight core controller.
- IPC communication: secure, local inter-process communication using Windows Named Pipes.
- Primary media formats: MP4, WebM, AVI.

### 3.3 Tier 3: Research Documents & Technical Context (Non-Mandatory Recommendations)
Technical proposals originating from research benchmarks (including Gemini research notes, Lively Wallpaper analysis, Wallpaper Engine comparative audits):
- WorkerW message `0x052C` window injection mechanics.
- libmpv vs. Windows Media Foundation evaluation.
- WebView2 for HTML5/WebGL rendering.
- MKV container format support.
- Numerical performance targets (e.g., idle RAM < 10 MB, idle CPU < 0.1%, pause latency < 50 ms).  
*Rule:* These recommendations are classified as `TECHNICAL-DECISION`, `RESEARCH-RECOMMENDATION`, or `UNRESOLVED`. They are never treated as user-approved requirements until empirically validated and formally accepted.

### 3.4 Tier 4: Missing or External Documents Disclosure
- **Audit Finding:** The original project brief and initial Gemini research documents are not present as independent markdown or text files in the workspace root or parent directories.
- **Remediation & Traceability Record:** Their full functional scopes, architectural propositions, format lists, performance goals, and operational parameters were directly provided in the comprehensive Step 1 user directive. Requirements derived from these specifications are recorded with their explicit derivation status to guarantee zero loss of project context.

---

## 4. Product Baseline Definition

| Attribute | Baseline Definition |
| :--- | :--- |
| **Product Name** | **KinetiWall** |
| **Product Category** | Windows Desktop Customization & Live Wallpaper Engine |
| **Target Platform** | Windows 10 (64-bit Version 2004 / Build 19041+) and Windows 11 (64-bit, all builds) |
| **Hardware Targets** | Desktop PCs, Workstations, and Battery-Powered Laptops (x86_64 architecture) |
| **Primary Language** | **Rust** (Edition 2021+, stable toolchain) |
| **Secondary Runtimes** | HTML5 / JavaScript / WebGL via Microsoft Edge WebView2 (for web wallpapers only) |
| **Product Purpose** | Provide a lightweight, fluid, visually rich, and rock-solid live wallpaper platform that transforms the Windows desktop with video, interactive web, and audio-reactive content without degrading gaming performance, battery life, or system responsiveness. |
| **Operating Model** | **Local-First & Completely Offline**: No mandatory cloud connections, accounts, external API dependencies, or background data exfiltration. |
| **Distribution Target** | Lightweight self-contained native installer with zero bloatware and clean uninstall capabilities. |

---

## 5. Scope Boundary & Anti-Goals (Explicit Non-Goals)

To prevent architectural drift and maintain technical focus, the following capabilities are explicitly declared as **Non-Goals** for the initial release baseline:

1. **KW-NG-001 (No Resident Gallery Background Process):** The gallery and settings UI must completely terminate its process when closed by the user. It must never linger as a hidden, RAM-consuming background process. Only the lightweight core daemon and active wallpaper players may run.
2. **KW-NG-002 (No Telemetry, Analytics, or Spyware):** The application must not transmit usage analytics, user behavioral telemetry, file metadata, or desktop screenshots to any remote endpoint.
3. **KW-NG-003 (No Mandatory Online Services or Cloud Accounts):** Core wallpaper playback, library management, and property customization must function without an internet connection or account creation.
4. **KW-NG-004 (No Direct Explorer Shell Modification or Hook Injection):** The engine must not inject DLLs into `explorer.exe` or hook undocumented internal Windows kernel APIs that compromise operating system stability.
5. **KW-NG-005 (No Permanent Audio Interception or Recording):** Audio visualization must process loopback PCM data strictly in transient volatile memory for FFT frequency calculation. It must never record, serialize to disk, or stream audio data.
6. **KW-NG-006 (No Unsandboxed Remote Executable Execution):** The application must never download and execute arbitrary unsigned executable binaries as wallpapers without sandboxing and explicit user confirmation.

---

## 6. Document Suite Organization

The requirements and governance documentation is organized into the following authoritative modules located in `docs/requirements/`:

1. **`AUDIT_AND_BASELINE.md` (KW-DOC-001):** Repository audit, toolchain verification, source precedence, product baseline, and governance.
2. **`REQUIREMENTS_INVENTORY.md` (KW-DOC-002):** Granular functional (`KW-F-xxx`) and non-functional (`KW-NF-xxx`) requirements inventory with observable acceptance criteria and verification protocols.
3. **`TRACEABILITY_MATRIX.md` (KW-DOC-003):** Full bidirectional 14-field traceability matrix cross-referencing sources, classifications, priorities, dependencies, and risks.
4. **`traceability_matrix.csv` (KW-DATA-001):** Machine-readable structured export of the traceability matrix for automated CI/agent validation.
5. **`UNRESOLVED_DECISIONS_REGISTER.md` (KW-DOC-004):** Detailed register of open architectural, technological, and compatibility decisions requiring research or empirical validation.
6. **`OPTIONAL_AND_FUTURE_FEATURES.md` (KW-DOC-005):** Catalog of candidate enhancements, post-v1 roadmap features, and speculative research suggestions.
7. **`README.md` (KW-DOC-000):** Directory index, requirements metrics summary, and navigation guide.

---

## 7. Change Control & Traceability Maintenance

Every subsequent development step (Steps 2 through 90) must adhere to these change-control protocols:
1. **Identifier Stability:** Existing requirement IDs (`KW-F-xxx`, `KW-NF-xxx`, `UDR-xxx`, `OFF-xxx`) are permanent and must not be renumbered or reassigned.
2. **Status Progression:** A requirement's status progresses strictly from `PLANNED` -> `IN_PROGRESS` -> `VERIFIED` -> `ACCEPTED` (or `DEFERRED` / `SUPERSEDED` upon approved change).
3. **No Silent Revisions:** Any alteration to acceptance criteria, priority, or technical approach requires updating the corresponding entries in `REQUIREMENTS_INVENTORY.md`, `TRACEABILITY_MATRIX.md`, and the change log.
4. **Implementation Gating:** An implementation step is considered verified only when its corresponding verification method passes observable testing under defined conditions.

