# KinetiWall — Technical Research Documentation Suite

**Document Identifier:** KW-RES-000  
**Phase:** 1 — Project Research, Requirements & Feasibility  
**Step:** 2 of 90 — Technical Research & Source Verification  
**Status:** Approved Technical Research Baseline  
**Date:** 2026-11-08  

---

## 1. Research Suite Overview

This directory (`docs/research/`) houses the complete engineering research and evidence verification deliverables produced during **Step 2 of 90** for **KinetiWall**.

All investigations rely on authoritative primary sources (Microsoft Learn, Rust-lang, Khronos, mpv.io, Snyk), maintainer issue reports (`rocksdanister/lively`, `Wallpaper Engine`), and empirical platform analysis. No unverified assumptions or marketing claims were accepted into the engineering baseline.

---

## 2. Navigational Index of Research Deliverables

| Deliverable | Identifier | Title & Description |
| :--- | :---: | :--- |
| **[`TECHNICAL_RESEARCH_REPORT.md`](./TECHNICAL_RESEARCH_REPORT.md)** | `KW-RES-001` | **Deliverable A:** Master Technical Research Report covering all 16 architectural domains (Desktop Integration, Rust Crates, Process Architecture, IPC, Video Engine, WebView2 Sandbox, Multi-Monitor, WASAPI Audio, Storage, UI, Threat Model, Offline Packaging, Competitor Audits, AI Feasibility, and Metrics Calibration). |
| **[`SOURCE_REGISTER.md`](./SOURCE_REGISTER.md)** | `KW-RES-002` | **Deliverable B:** Primary Source Evidence Register cataloging 10 foundational primary sources (`SR-001` to `SR-010`) with direct URLs, access dates, evidence established, limitations, confidence ratings, and architectural consequences. |
| **[`TECHNICAL_DECISION_COMPARISON.md`](./TECHNICAL_DECISION_COMPARISON.md)** | `KW-RES-003` | **Deliverable C:** Architectural Trade-off Analysis evaluating candidate technologies across 10 consistent engineering dimensions (Windows bindings, process boundaries, video decoders, licensing, IPC, UI, occlusion, web sandbox, database, and AI policy). |
| **[`RISK_AND_VALIDATION_REGISTER.md`](./RISK_AND_VALIDATION_REGISTER.md)** | `KW-RES-004` | **Deliverable D:** Technical Risk Assessment & Proof-of-Concept Protocol identifying 8 high-consequence platform risks (`TR-001` to `TR-008`), mitigation strategies, and concrete PoC exit criteria for subsequent development steps. |
| **[`RESEARCH_TRACEABILITY.md`](./RESEARCH_TRACEABILITY.md)** | `KW-RES-005` | **Deliverable E:** Research-to-Requirements Traceability Matrix mapping 20 research conclusions (`RC-001` to `RC-020`) directly to Step 1 requirement IDs (`KW-F`, `KW-NF`, `KW-NG`, `UDR`, `OFF`), and formally disproving 4 outdated prior assumptions. |

---

## 3. Key Research Conclusions & Verified Facts

```
+---------------------------------------------------------------------------------------------------+
|                              SUMMARY OF VERIFIED RESEARCH FINDINGS                                |
+-----------------------------+---------------------------------------------------------------------+
| Domain                      | Empirical Engineering Conclusion                                    |
+-----------------------------+---------------------------------------------------------------------+
| Windows Desktop Integration | Undocumented 0x052C message creates WorkerW, but Windows 11 24H2     |
|                             | breaks WorkerW when desktop icons are hidden. Dual-mode fallback    |
|                             | parenting (WorkerW vs direct Progman) is required.                  |
+-----------------------------+---------------------------------------------------------------------+
| Rust Multi-Process Engine   | Dedicated binaries (core daemon, UI, players) preserve <10 MB idle  |
|                             | RAM. Windows Job Objects (KILL_ON_JOB_CLOSE) eliminate orphans.     |
+-----------------------------+---------------------------------------------------------------------+
| Local IPC Security          | Windows Named Pipes with DACL restricted strictly to current user   |
|                             | SID prevent cross-session hijacking with zero firewall alerts.      |
+-----------------------------+---------------------------------------------------------------------+
| Video Playback Engine       | Embedded libmpv-2.dll with d3d11va and vo=gpu-next achieves gapless |
|                             | looping without 1-frame black flashes. Dynamic LGPL 2.1 linking     |
|                             | satisfies licensing without copyleft GPL contamination.            |
+-----------------------------+---------------------------------------------------------------------+
| Interactive Web Wallpapers  | Edge WebView2 using SetVirtualHostNameToFolderMapping isolates web   |
|                             | scripts from host filesystem while supporting relative assets.      |
+-----------------------------+---------------------------------------------------------------------+
| Fullscreen & Occlusion      | SetWinEventHook foreground tracking achieves <0.05% CPU overhead,   |
|                             | combined with SHQueryUserNotificationState for game detection.      |
+-----------------------------+---------------------------------------------------------------------+
| Audio Visualization Engine  | WASAPI loopback capture with rustfft processes audio strictly in    |
|                             | volatile memory; IMMNotificationClient handles headphone switching. |
+-----------------------------+---------------------------------------------------------------------+
| Desktop Gallery UI          | Tauri v2 on-demand process completely terminates on window close,   |
|                             | fulfilling KW-NG-001 (Zero resident UI background memory).         |
+-----------------------------+---------------------------------------------------------------------+
| AI / SLM Feasibility        | Local neural models consume 2.5–5.0 GB VRAM and starve GPU gaming   |
|                             | performance; strictly deferred to external companion tooling.       |
+-----------------------------+---------------------------------------------------------------------+
```

---

## 4. Transition to Phase 1, Step 3 (Windows Desktop Architecture Spike)

Step 2 has established the necessary factual and evidentiary baseline. The immediate next action in the KinetiWall roadmap is:
- **Step 3:** Implement an isolated, experimental Win32 desktop integration spike prototyping `Progman` / `WorkerW` message `0x052C` attachment, handle recovery on `TaskbarCreated`, icon hit-testing, and verify dual-mode fallback behavior on Windows 11 24H2 in compliance with `TR-001` and `UDR-002`.

