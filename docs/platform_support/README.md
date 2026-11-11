# KinetiWall — Windows Platform Support Policy Documentation Suite

**Document Identifier:** KW-POL-000  
**Phase:** 1 — Project Research, Requirements & Feasibility  
**Step:** 4 of 90 — Windows Compatibility & Platform Support Policy  
**Status:** Approved Platform Policy Baseline  
**Date:** 2026-11-08  

---

## 1. Platform Policy Suite Overview

This directory (`docs/platform_support/`) houses the complete engineering platform support policy, Windows compatibility matrices, integration risk assessments, runtime dependency analyses, verification test protocols, and governance registers produced during **Step 4 of 90** for **KinetiWall**.

All policies are grounded in authoritative primary evidence (Microsoft Learn, Windows Release Health dashboards, Microsoft Lifecycle datasheets, and Rust toolchain specifications). The policy honors the user's original requirements while reconciling them with modern operating system servicing realities.

---

## 2. Navigational Index of Platform Support Deliverables

| Deliverable | Identifier | Title & Description |
| :--- | :---: | :--- |
| **[`WINDOWS_SUPPORT_POLICY.md`](./WINDOWS_SUPPORT_POLICY.md)** | `KW-POL-001` | **Deliverable A:** Master Windows Platform Support Policy establishing the five-tier support taxonomy (`Tier A`, `Tier B`, `Tier C`, `Tier D`, `Tier U`), hardware prerequisites, lifecycle distinctions, exclusion rules, and capability detection principles. |
| **[`WINDOWS_COMPATIBILITY_MATRIX.md`](./WINDOWS_COMPATIBILITY_MATRIX.md)** | `KW-POL-002` | **Deliverable B:** Evidence-Backed Compatibility Matrix evaluating Windows 11 (24H2, 23H2, 22H2, 21H2), Windows 10 (22H2 through 1903/1809 LTSC), specialized Windows editions (N/KN, IoT, Server), and processor architectures. |
| **[`DESKTOP_INTEGRATION_RISK_REGISTER.md`](./DESKTOP_INTEGRATION_RISK_REGISTER.md)** | `KW-POL-003` | **Deliverable C:** Systems-Level Desktop Integration Risk Register distinguishing documented Win32 APIs from undocumented shell behaviors (`0x052C`, `WorkerW`), detailing the Windows 11 24H2 hidden-icons composition quirk, and defining the dual-mode attachment algorithm. |
| **[`RUNTIME_DEPENDENCY_MATRIX.md`](./RUNTIME_DEPENDENCY_MATRIX.md)** | `KW-POL-004` | **Deliverable D:** Runtime & Dependency Compatibility Matrix evaluating Microsoft Edge WebView2 (Evergreen vs Fixed Version), `libmpv-2.dll` (Direct3D 11 video acceleration, LGPL 2.1 compliance), WASAPI audio, Direct3D 11 feature levels, and MSVC CRT runtimes. |
| **[`COMPATIBILITY_TEST_MATRIX.md`](./COMPATIBILITY_TEST_MATRIX.md)** | `KW-POL-005` | **Deliverable E:** Formal 20-Scenario Verification Test Protocol (`CT-001` to `CT-020`) defining execution prerequisites, steps, expected outcomes, and support-blocking failure severities (status: `NOT RUN (PLANNED)`). |
| **[`UNRESOLVED_PLATFORM_DECISIONS.md`](./UNRESOLVED_PLATFORM_DECISIONS.md)** | `KW-POL-006` | **Deliverable F:** Unresolved Platform Decision Register analyzing 12 core platform policy decisions (`DEC-PLT-001` to `DEC-PLT-012`), including Windows 10 1903 reconciliation, post-2025 ESU commitments, and dual-distribution packaging. |
| **[`PLATFORM_SOURCE_REGISTER.md`](./PLATFORM_SOURCE_REGISTER.md)** | `KW-POL-007` | **Deliverable G:** Bibliographic Primary Source Evidence Register cataloging verified sources (`PSR-001` to `PSR-008`) with direct URLs, access dates (2026-11-08), evidence established, and project relevance. |

---

## 3. High-Level Windows Support Dashboard

```
+---------------------------------------------------------------------------------------------------+
|                                  KINETIWALL PLATFORM SUPPORT MATRIX                               |
+--------------------------+-----------------------+---------------------+--------------------------+
| Operating System Release | Architecture          | Support Tier        | Servicing / Support Note |
+--------------------------+-----------------------+---------------------+--------------------------+
| Windows 11 24H2 (26100)  | x86-64 (AMD64)        | **Tier A (Primary)**| Dual-mode shell attachment required for hidden icons. |
| Windows 11 23H2 (22631)  | x86-64 (AMD64)        | **Tier A (Primary)**| Recommended baseline. Full Direct3D 11 & WebView2.   |
| Windows 11 22H2 (22621)  | x86-64 (AMD64)        | **Tier A (Primary)**| Enterprise servicing. Highly stable.                  |
| Windows 10 22H2 (19045)  | x86-64 (AMD64)        | **Tier A (Primary)**| Supported through October 2027 (Consumer ESU period).|
| Windows 10 21H2 LTSC     | x86-64 (AMD64)        | **Tier A (Primary)**| Enterprise LTSC supported through January 2027.      |
| Windows 10 1903 / 1909   | x86-64 (AMD64)        | **Tier B (Limited)**| Preserves user requirement; manual WebView2 needed.  |
| Windows 11 on ARM64      | x86-64 (Prism Emul.)  | **Tier B (Limited)**| Snapdragon X devices supported via Prism translation.|
| Windows 11 on ARM64      | Native (aarch64-msvc) | **Tier C (Target)** | Experimental roadmap target (Phase 2 cross-compile). |
| Windows 7 / 8 / 8.1      | Any Architecture      | **Tier D (Excluded)**| Lacks required DXGI, DWM, and WebView2 runtimes.     |
| Windows 10 / 11 32-bit   | x86 (IA-32)           | **Tier D (Excluded)**| Excluded due to 2 GB memory address space limits.     |
| Windows Server (All)     | Any Architecture      | **Tier D (Excluded)**| Desktop Experience and Windows Audio disabled.        |
+--------------------------+-----------------------+---------------------+--------------------------+
```

---

## 4. Key Takeaways for Step 5 (Windows Desktop Attachment Spike)

Step 4 has established the official Windows platform support policy and identified the critical technical gating items:
1. **Physical Validation of Dual-Mode Attachment:** Step 5 must implement an isolated Rust Win32 prototype executing `0x052C` WorkerW injection and direct `Progman` parenting to verify stability on Windows 11 24H2 with desktop icons both enabled and disabled (`TR-001`, `CT-004`, `CT-005`).
2. **Explorer Restart Recovery Testing:** Verify interception of `TaskbarCreated` with a 500ms debounce timer during active playback (`CT-006`).
3. **No Codebase Contamination:** Step 4 has produced documentation exclusively under `docs/platform_support/`. Application source code remains completely untouched.

