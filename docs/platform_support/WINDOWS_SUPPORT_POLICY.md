# KinetiWall — Windows Platform Support Policy

**Document Identifier:** KW-POL-001  
**Document Type:** Formal Engineering Platform Support Policy (Deliverable A)  
**Current Phase:** Phase 1 — Project Research, Requirements & Feasibility  
**Step:** Step 4 of 90 — Windows Compatibility & Platform Support Policy  
**Status:** Approved Support Policy Baseline  
**Date of Research:** 2026-11-08  
**Author:** Principal Windows Platform Architect & Systems Engineering Team  

---

## 1. Executive Summary & Scope

This document formalizes the official **Windows Compatibility & Platform Support Policy** for **KinetiWall**, a native Windows desktop customization and live wallpaper engine engineered in **Rust**.

KinetiWall requires deep, reliable integration with the Windows desktop shell (`explorer.exe`), the Desktop Window Manager (DWM), Direct3D 11 hardware-accelerated presentation pipelines, Microsoft Edge WebView2 controls, the Windows Audio Session API (WASAPI), and low-overhead kernel event hooks (`SetWinEventHook`).

To ensure that KinetiWall delivers rock-solid stability without silent regressions, broken desktop layers, or runaway maintenance costs, this policy:
1. Establishes a five-tier support taxonomy (`Tier A`, `Tier B`, `Tier C`, `Tier D`, `Tier U`).
2. Reconciles the user's original requirement (Windows 10 Version 1903+) with modern Windows OS servicing baselines.
3. Defines the processor architecture strategy across **x86-64 (AMD64)** and **ARM64** (native vs. Prism emulation).
4. Separates technical compatibility from official product support commitments.
5. Prescribes capability detection and graceful failure recovery protocols.

---

## 2. Authoritative Requirements & Precedence Reconciliation

In strict adherence to Step 4 governance rules, user-requested platform targets are preserved as authoritative baselines.

```
+---------------------------------------------------------------------------------------------------+
|                            REQUIREMENTS PRECEDENCE & COMPATIBILITY                                |
+---------------------------------------------------------------------------------------------------+
| TIER 1: ORIGINAL USER BRIEF (AUTHORITATIVE)                                                       |
| - Target Operating Systems: Windows 10 (64-bit Version 1903 or higher) & Windows 11               |
| - Target Architectures: x86-64 and ARM64                                                          |
+---------------------------------------------------------------------------------------------------+
                                  |
                                  v (Reconciled via Step 4 Empirical Research)
+---------------------------------------------------------------------------------------------------+
| TIER 2: RECONCILED PLATFORM POLICY (PROPOSED ENGINEERING BASELINE)                                |
| - Tier A (Primary Supported):                                                                     |
|     * Windows 11 (64-bit): 24H2 (Build 26100), 23H2 (Build 22631), 22H2 (Build 22621)             |
|     * Windows 10 (64-bit): 22H2 (Build 19045) / 21H2 (Build 19044) / 2004 (Build 19041)           |
|     * Architecture: x86-64 (AMD64)                                                                |
| - Tier B (Supported with Limitations):                                                            |
|     * Windows 10 (64-bit): Versions 1903 (Build 18362) and 1909 (Build 18363)                      |
|       (Limitation: Requires manual WebView2 runtime installation; ESU security notices apply)     |
|     * Windows 11 on ARM64 via Prism Emulation (x64 binary emulated on Snapdragon X Elite/Plus)   |
| - Tier C (Experimental / Roadmap Target):                                                         |
|     * Windows 11 Native ARM64 (aarch64-pc-windows-msvc)                                           |
|       (Limitation: Requires cross-compiling libmpv-2.dll and physical ARM64 testbench)             |
| - Tier D (Unsupported / Excluded):                                                                |
|     * Windows 7, 8.0, 8.1; Windows 10 32-bit (x86); Windows Server; Windows 10X                  |
+---------------------------------------------------------------------------------------------------+
```

### 2.1 The Windows 10 Version 1903 vs. 2004 Reconciliation (`DEC-PLT-001`)
- **Original User Requirement:** Support Windows 10 64-bit Version 1903 or higher.
- **Empirical Platform Finding:**
  - Microsoft terminated standard lifecycle support for Windows 10 Version 1903 on **December 8, 2020** (`PSR-002`).
  - Windows 10 Version 2004 (Build 19041) introduced a unified servicing architecture. Builds 19041 (2004), 19042 (20H2), 19043 (21H1), 19044 (21H2), and 19045 (22H2) share an identical kernel, Win32 API layer, and graphics subsystem.
  - Microsoft Edge WebView2's virtual host mapping (`ICoreWebView2_3::SetVirtualHostNameToFolderMapping`) and Per-Monitor V2 DPI synchronization are officially validated on Build 19041+.
  - On **October 14, 2025**, Microsoft retired standard support for Windows 10 entirely, transitioning consumer and enterprise devices onto Extended Security Updates (ESU) through October 2027/2028 (`PSR-003`).
- **Policy Reconciliation:** Windows 10 Version 1903 is **not silently removed**. It is classified as **Tier B: Supported with Limitations**, while Windows 10 Build 19041+ (through 22H2) is designated as **Tier A: Primary Supported**. This preserves user intent while providing an honest engineering assessment of platform maintenance.

---

## 3. Formal Support Tier Taxonomy

Every combination of Windows operating system release, edition, and processor architecture is classified into one of five mutually exclusive support tiers:

### Tier A: Primary Supported
- **Definition:** The official, guaranteed target environment for KinetiWall production releases.
- **Criteria:**
  1. Operating system is within active Microsoft mainstream or extended security servicing.
  2. All core runtime dependencies (Direct3D 11, `libmpv-2.dll`, WebView2, WASAPI) are natively supported.
  3. All 20 compatibility test scenarios (`CT-001` to `CT-020`) pass automated CI verification.
  4. Software regressions on Tier A platforms are treated as Release-Blocking (P0) defects.

### Tier B: Supported with Limitations
- **Definition:** Fully functional environments where specific platform limitations, discontinued vendor servicing, or third-party dependencies impose documented functional constraints.
- **Criteria:**
  1. Core wallpaper playback and desktop integration function safely.
  2. Documented limitations exist (e.g. requires manual WebView2 installation, or operates under instruction-set emulation).
  3. The installer and application detect the environment and surface actionable, non-intrusive notices.
  4. Tested periodically; non-critical edge-case bugs receive P2 maintenance priority.

### Tier C: Experimental / Compatibility-Targeted
- **Definition:** Architectures or upcoming Windows Insider preview builds under active technical investigation but not yet backed by complete automated test benches.
- **Criteria:**
  1. Technical feasibility is indicated by research, but complete end-to-end testing has not been executed on physical hardware.
  2. Binaries may be provided for community evaluation with explicit "Experimental" labeling.
  3. Never advertised as a release-ready production guarantee.

### Tier D: Unsupported / Formally Excluded
- **Definition:** Environments that are outside product scope, technologically incapable of supporting the architecture, or pose unacceptable security hazards.
- **Criteria:**
  1. Lacks required Win32 APIs (e.g. Windows 7 lacks modern WebView2 and DXGI swapchain features).
  2. Missing 64-bit memory addressing (all 32-bit x86 OSs).
  3. The installer detects the platform, halts execution with a user-friendly error message, and exits safely.

### Tier U: Unknown / Unverified
- **Definition:** Specialized Windows distributions (e.g. customized lite ISOs, third-party shell replacements) where empirical test evidence is absent.
- **Criteria:**
  1. The application attempts standard capability detection and executes fallback routines.
  2. No support guarantees are provided until verified.

---

## 4. Minimum & Recommended System Prerequisites

| Hardware / Subsystem Component | Minimum Functional Requirement (Baseline) | Recommended Target (Optimal Fluidity) |
| :--- | :--- | :--- |
| **Operating System** | Windows 10 64-bit Version 1903 (Build 18362) [Tier B] | Windows 11 64-bit Version 23H2 / 24H2 [Tier A] |
| **Primary Baseline OS** | Windows 10 64-bit Version 2004 (Build 19041+) [Tier A]| Windows 11 Version 24H2 (Build 26100+) [Tier A] |
| **Processor (CPU)** | x86-64 Dual-Core 1.8 GHz (Intel 6th Gen / AMD Ryzen 1000) | x86-64 Quad-Core 2.5 GHz+ (Intel 8th Gen+ / Ryzen 3000+) |
| **System Memory (RAM)** | 2.0 GB System RAM | 8.0 GB+ System RAM (DDR4 / DDR5) |
| **Graphics Adapter (GPU)** | Direct3D 11.0 Feature Level 10_0 (Intel HD Graphics 4000) | Direct3D 11.1+ Feature Level 11_1 (NVIDIA GTX 10-series / AMD RX 500+ / Intel Iris Xe) |
| **Hardware Video Decoder** | DXVA2 / D3D11VA supporting H.264 / AVC 1080p | Dedicated Video Engine (NVDEC / VCN / QuickSync) supporting 4K H.264, VP9, AV1 |
| **Audio Subsystem** | Windows Core Audio / WASAPI compatible audio endpoint | Dedicated audio playback device (44.1 kHz / 48.0 kHz 32-bit float) |
| **Display Configuration** | 1280×720 at 60 Hz (Standard DPI 100%) | Multi-Monitor 1080p / 1440p / 4K (120 Hz+ High Refresh Rate, Mixed DPI) |
| **Storage (Disk Space)** | 100 MB available storage (excluding user wallpapers) | 1.0 GB+ SSD storage (for high-speed thumbnail caching) |
| **Web Runtime** | Microsoft Edge WebView2 Runtime (Evergreen) | Preinstalled Evergreen WebView2 (Standard on Windows 11 and Win 10 2004+) |

---

## 5. Operating System Lifecycle Distinction

To avoid conflating vendor servicing with software execution capabilities, KinetiWall formalizes five distinct lifecycle categories:

```
+---------------------------------------------------------------------------------------------------+
|                              LIFECYCLE & SUPPORT CATEGORIZATION                                   |
+---------------------------------------------------------------------------------------------------+
| 1. Technically Compatible: The OS kernel and user32.dll export all required APIs. The binary runs.|
| 2. Successfully Tested: Verified by KinetiWall automated test runs on physical/VM test benches.  |
| 3. Officially Supported: KinetiWall issues patches, validates CI, and guarantees SLA for bugs.   |
| 4. In Microsoft Servicing: Receives regular security updates directly from Windows Update.       |
| 5. In Extended Servicing (ESU): Covered under paid/consumer Extended Security Updates (2025–2028).|
+---------------------------------------------------------------------------------------------------+
```

- **Policy Principle:** A platform being *technically compatible* does not obligate KinetiWall to provide *official support*. Official support is granted only when automated CI validation is maintained and vendor platform stability is assured.
- **Windows 10 Post-2025 Reality:** Although Microsoft ended standard free servicing for Windows 10 on October 14, 2025, over 40% of desktop workstations globally remain on Windows 10 22H2 under consumer and enterprise ESU. KinetiWall **will officially support Windows 10 22H2 through at least October 2027**, aligning with Microsoft's consumer ESU milestone (`PSR-003`).

---

## 6. Processor Architecture Strategy

### 6.1 x86-64 (AMD64) — Initial Release Sole Primary Architecture
- **Status:** **Tier A: Primary Supported**
- **Justification:** x86-64 accounts for >92% of the active Windows PC gaming and workstation ecosystem. The Rust toolchain (`x86_64-pc-windows-msvc`), Direct3D 11 drivers, `libmpv-2.dll`, WebView2, and audio libraries are mature, stable, and proven.
- **Commitment:** All v1.0 features must achieve 100% test pass on x86-64 before general availability.

### 6.2 ARM64 (Snapdragon X Elite / Plus / Surface Pro) — Phased Strategy
- **Prism Emulation (Windows 11 24H2):** **Tier B: Supported with Limitations**
  - Windows 11 24H2 introduced Microsoft's **Prism emulation engine**, which translates x64 instructions to ARM64 dynamically with support for AVX/AVX2 vector extensions (`PSR-004`).
  - KinetiWall's x86-64 binaries run under Prism on Snapdragon X devices. However, because video decoding and DWM window parenting incur minor emulation overhead, it is classified as Tier B.
- **Native ARM64 (`aarch64-pc-windows-msvc`):** **Tier C: Experimental / Post-v1 Roadmap**
  - Compiling pure Rust binaries for ARM64 is trivial (`cargo build --target aarch64-pc-windows-msvc`).
  - *The Blocking Dependency:* Pre-built native ARM64 builds of `libmpv-2.dll` and FFmpeg shared libraries are not readily standardized under official LGPL 2.1 distribution channels, requiring custom cross-compilation toolchains. Native ARM64 is scheduled for Phase 2 feasibility spikes (`UDR-001`, `DEC-PLT-004`).

---

## 7. Windows Editions & Servicing Considerations

| Windows Edition Family | Support Tier | Technical & Operational Considerations |
| :--- | :---: | :--- |
| **Windows 11 Home & Pro** | **Tier A** | Standard consumer target. Evergreen WebView2 preinstalled. Automatic DWM composition. |
| **Windows 11 Enterprise & Education**| **Tier A** | Fully supported. Standard enterprise Group Policies supported under least privilege (`KW-NF-011`). |
| **Windows 10 Home & Pro (2004–22H2)**| **Tier A** | Standard target. Fully compatible with Direct3D 11 and WebView2. Supported through Oct 2027. |
| **Windows 10 Enterprise LTSC 2021** | **Tier A** | Built on Build 19044 (21H2). Long-term servicing supported by Microsoft through 2027/2031. |
| **Windows 10 Enterprise LTSC 2019** | **Tier B** | Built on Build 17763 (1809). Pre-dates modern WebView2; requires standalone runtime installer. |
| **Windows 10 Versions 1903 / 1909** | **Tier B** | User-requested baseline. Requires manual WebView2 installation; Microsoft lifecycle expired. |
| **Windows N & KN Editions** | **Tier B** | Lacks Windows Media Feature Pack by default. Video engine will fail unless user installs Media Pack. |
| **Windows Server (2019, 2022, 2025)** | **Tier D** | Excluded. Desktop Experience, audio endpoints, and DWM composition are typically disabled. |
| **Windows 10 / 11 32-bit (x86)** | **Tier D** | Excluded. Memory addressing limitations prevent stable high-resolution video decoding. |

---

## 8. Capability Detection & Graceful Fallback Principles

To avoid brittle, hardcoded OS build checks that break across minor Windows patches, KinetiWall adheres to the **Feature Capability Detection Principle**:

1. **Window Attachment Validation:** The engine attempts sub-desktop attachment via `0x052C` dynamically. If `WorkerW` is not detected within 500 ms (e.g. on Windows 11 24H2 with hidden icons), it executes the direct `Progman` parenting fallback automatically without checking build numbers (`KW-F-003`).
2. **Hardware Decoding Fallback:** When loading video media, the player queries Direct3D 11 video device capabilities (`ID3D11VideoDevice`). If hardware decoding context creation fails, it falls back seamlessly to software CPU decoding and logs a warning (`KW-F-015`).
3. **WebView2 Runtime Detection:** The installer and web player query the Windows Registry for the Evergreen runtime GUID. If absent, the UI informs the user with an actionable download link rather than crashing (`UDR-006`).
4. **Media Feature Pack Detection:** On Windows N/KN editions, the core checks for `mfplat.dll`. If missing, a clear UI dialog instructs the user to enable the optional Windows Media Feature Pack.

---

## 9. Policy Change Management & Long-Term Governance

Platform support commitments are governed by strict engineering change protocols:
1. **Windows Insider Preview Tracking:** Upcoming Windows feature updates are evaluated in the Windows Insider Beta and Release Preview rings at least 90 days prior to general availability.
2. **Deprecation Policy:** A supported operating system version can only be deprecated after:
   - Microsoft officially terminates all standard and ESU servicing for that release.
   - An advance deprecation notice is displayed in KinetiWall release notes for at least one minor release cycle (e.g. 60 days).
   - The user/product owner formally approves the policy revision.
3. **No Retroactive Removal:** User-approved baselines will never be silently dropped to satisfy developer convenience.

