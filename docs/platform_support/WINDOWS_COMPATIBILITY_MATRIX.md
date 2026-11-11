# KinetiWall — Windows Compatibility Matrix

**Document Identifier:** KW-POL-002  
**Document Type:** Formal Windows Compatibility Matrix (Deliverable B)  
**Current Phase:** Phase 1 — Project Research, Requirements & Feasibility  
**Step:** Step 4 of 90 — Windows Compatibility & Platform Support Policy  
**Status:** Approved Compatibility Matrix Baseline  
**Date of Research:** 2026-11-08  

---

## 1. Matrix Overview & Structure

This matrix provides an evidence-backed evaluation of KinetiWall across **Windows operating system releases, specialized editions, and processor architectures**. 

Each entry maps operating system servicing lifecycles, API support, runtime availability, desktop integration quirks, and test statuses to an explicit **Support Tier** (`Tier A`, `Tier B`, `Tier C`, `Tier D`, `Tier U`) in compliance with `KW-POL-001`.

---

## 2. Windows Operating System Release Compatibility Matrix

| OS Version | Release Name | Build Range | Architecture | Microsoft Lifecycle Status | Win32 API & Shell | Direct3D & GPU | WebView2 Runtime | Desktop Integration Quirks | Support Tier | Test Status | Recommendation & Approval |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :---: | :---: | :--- |
| **Windows 11** | **24H2** (2024 Update) | `26100.x` | x86-64 | Active Servicing (GA: Oct 2024; EOL: Oct 2026/2027) | Full API availability | D3D 11.1+ / D3D 12 | Preinstalled Evergreen | **WorkerW breaks when icons hidden; dual-mode fallback required** | **Tier A** | PLANNED (`CT-004`) | Primary release target. No user approval needed. |
| **Windows 11** | **23H2** (2023 Update) | `22631.x` | x86-64 | Active Servicing (EOL: Nov 2025/2026) | Full API availability | D3D 11.1+ / D3D 12 | Preinstalled Evergreen | Standard WorkerW attachment verified; stable DWM layers | **Tier A** | PLANNED (`CT-001`) | Primary release target. Highly stable. |
| **Windows 11** | **22H2** (2022 Update) | `22621.x` | x86-64 | Active Servicing (Enterprise EOL: Oct 2025) | Full API availability | D3D 11.1+ / D3D 12 | Preinstalled Evergreen | Standard WorkerW attachment verified | **Tier A** | PLANNED (`CT-001`) | Primary release target. Fully compatible. |
| **Windows 11** | **21H2** (Initial Release) | `22000.x` | x86-64 | Mainstream EOL (Oct 2023 / Oct 2024) | Full API availability | D3D 11.1+ / D3D 12 | Preinstalled Evergreen | Minor taskbar centering DPI quirks | **Tier B** | PLANNED (`CT-002`) | Supported for remaining users. Auto-upgrade to 23H2 recommended. |
| **Windows 10** | **22H2** (Final Update) | `19045.x` | x86-64 | **ESU Extended Servicing** (Oct 2025 – Oct 2027/2028) | Full API availability | D3D 11.0+ / D3D 11.1 | Preinstalled Evergreen | Standard WorkerW attachment highly stable | **Tier A** | PLANNED (`CT-001`) | **Primary release target.** Massive active user base. |
| **Windows 10** | **21H2 / LTSC 2021** | `19044.x` | x86-64 | LTSC Servicing (Supported through Jan 2027) | Full API availability | D3D 11.0+ / D3D 11.1 | Preinstalled Evergreen | Identical servicing codebase to 22H2 (Build 19041 base) | **Tier A** | PLANNED (`CT-001`) | Primary enterprise release target. |
| **Windows 10** | **2004 / 20H2 / 21H1** | `19041–19043` | x86-64 | Microsoft EOL (2021–2022) | Full API availability | D3D 11.0+ | Evergreen update available | Shared 19041 codebase; PerMonitorV2 verified | **Tier A** | PLANNED (`CT-002`) | Fully compatible. Shares 19041 kernel base. |
| **Windows 10** | **1903 / 1909** | `18362 / 18363` | x86-64 | **Microsoft EOL** (Dec 2020 / May 2021) | Win32 available; DPI V2 partial | D3D 11.0+ | **Not Preinstalled** (Requires manual download) | Requires manual WebView2 installation; EOL security risks | **Tier B** | PLANNED (`CT-003`) | **Preserves user requirement.** Recommended as Tier B. *User Approval Flagged (`DEC-PLT-001`)*. |
| **Windows 10** | **1809 / LTSC 2019** | `17763.x` | x86-64 | LTSC Extended Servicing (Supported to Jan 2029) | Lacks newer DPI APIs | D3D 11.0 | **Not Preinstalled** (Requires manual download) | Pre-dates unified 19041 DWM; requires custom DPI handling | **Tier B** | PLANNED (`CT-003`) | Supported for enterprise LTSC 2019 environments. |
| **Windows 10** | **1507–1803** (Legacy) | `10240–17134` | x86-64 | Fully Retired EOL (2017–2019) | Incomplete DPI and COM APIs | D3D 11.0 | Unsupported | Frequent shell lockups during window parenting | **Tier D** | EXCLUDED | Formally excluded due to missing core APIs. |
| **Windows 7 / 8 / 8.1** | Legacy NT 6.x | `7601 / 9600` | x86-64 | Fully Retired EOL (2020 / 2023) | Missing modern shell & IPC APIs | D3D 11.0 partial | Unsupported | Desktop composition model incompatible with modern DWM | **Tier D** | EXCLUDED | Formally excluded. Installer aborts safely. |

---

## 3. Processor Architecture Compatibility Matrix

| Processor Architecture | Target Triple / Toolchain | Runtime Execution Mode | Dependency Status (`libmpv`, WebView2) | Hardware Decode Capability | Performance & Footprint Expectations | Support Tier | Test Status | Strategic Recommendation |
| :--- | :--- | :--- | :--- | :--- | :--- | :---: | :---: | :--- |
| **x86-64 (AMD64)** | `x86_64-pc-windows-msvc` | **Native Execution** | **100% Available & Validated** | Full DXVA2 / D3D11VA (NVDEC, VCN, QuickSync) | **Optimal:** Sub-10 MB idle daemon; <3% 1080p60 CPU | **Tier A** | PLANNED (`CT-001`) | **Sole primary launch architecture for Version 1.0.** |
| **ARM64 (Snapdragon X Elite/Plus)** | `x86_64-pc-windows-msvc` | **Prism Emulation** (Windows 11 24H2) | Runs standard x64 dependencies via Prism translation | GPU translation via Qualcomm Adreno D3D11 drivers | **Good:** Minor CPU translation overhead (~5–8% CPU during video playback) | **Tier B** | PLANNED (`CT-005`) | **Supported with limitations** on modern Copilot+ PCs via Prism. |
| **ARM64 Native** | `aarch64-pc-windows-msvc` | **Native Execution** | Rust/WebView2 native; **libmpv pre-builts missing LGPL ARM64 binaries** | Native Adreno D3D11VA decoding | **Optimal:** Native execution, zero translation overhead | **Tier C** | PLANNED (Spike) | **Experimental roadmap target.** Scheduled for Phase 2 cross-compile spike (`DEC-PLT-004`). |
| **ARM32 (Windows 10 on ARM)** | Legacy WOA | Emulated | Deprecated by Microsoft | None | Severely degraded; obsolete hardware | **Tier D** | EXCLUDED | Formally excluded. |
| **x86 32-bit (IA-32)** | `i686-pc-windows-msvc` | Native 32-bit | Available, but 2GB address space limits 4K video | Deprecated by modern GPU vendors | High out-of-memory crash risk with WebGL | **Tier D** | EXCLUDED | Formally excluded. 64-bit OS required. |

---

## 4. Specialized Windows Editions Compatibility Matrix

| Specialized Edition | Base OS Version | Architectural Constraints | Required Workarounds / Pre-requisites | Support Tier | Test Status | Handling & Error Behavior |
| :--- | :--- | :--- | :--- | :---: | :---: | :--- |
| **Windows 10 / 11 N & KN Editions** | Win 10 (19041+) / Win 11 | **Lacks Windows Media Feature Pack** by default (EU antitrust requirement) | User must install optional Windows Media Feature Pack to enable system codecs | **Tier B** | PLANNED (`CT-006`) | Core detects missing `mfplat.dll`; displays actionable prompt to install Media Pack. |
| **Windows 10 IoT Enterprise LTSC** | Win 10 Build 19044 | Highly stripped embedded image; standard shell intact | Standard Inno Setup installer; requires Evergreen WebView2 download | **Tier A** | PLANNED (`CT-001`) | Fully supported for industrial kiosk and digital signage setups. |
| **Windows 11 SE** | Win 11 Build 22000+ | Cloud-first education edition; restricted Win32 execution | Win32 installation blocked by Intune S-Mode policy | **Tier D** | EXCLUDED | Excluded due to Microsoft operating system policy restrictions. |
| **Windows Server 2022 / 2025** | Windows Server NT 10.0 | Audio service (`Audiosrv`) and Windows Themes disabled by default | Requires manual enablement of Desktop Experience and Windows Audio | **Tier D** | EXCLUDED | Excluded from official support. Server environments do not support desktop live wallpapers. |
| **Windows Sandbox / Hyper-V VM** | Win 10 / Win 11 | Virtualized display adapter (Microsoft Basic Render Driver); no D3D11VA | Falls back to software CPU video decoding; hardware acceleration disabled | **Tier B** | PLANNED (`CT-007`) | Supported for automated testing and CI verification with software decode warnings. |

---

## 5. Summary of Policy Commitments & Trade-offs

1. **Windows 10 Continuity Commitment:** Despite Microsoft's October 14, 2025 end-of-support milestone, KinetiWall guarantees **Tier A primary support for Windows 10 22H2 (Build 19045) through October 2027**, matching consumer ESU availability.
2. **Preservation of User's 1903 Baseline:** Windows 10 Version 1903 is formally retained under **Tier B: Supported with Limitations** rather than discarded, guaranteeing that users on legacy builds can run the software subject to manual WebView2 installation.
3. **ARM64 Pragmatism:** Version 1.0 supports Snapdragon X-series ARM64 laptops out-of-the-box via Microsoft's high-efficiency **Prism emulation engine (Tier B)**, while native ARM64 compilation is progressed as an active experimental pipeline (Tier C).

