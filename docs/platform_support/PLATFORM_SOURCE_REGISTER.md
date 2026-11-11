# KinetiWall — Platform Source Evidence Register

**Document Identifier:** KW-POL-007  
**Document Type:** Formal Platform Source Evidence Register (Deliverable G)  
**Current Phase:** Phase 1 — Project Research, Requirements & Feasibility  
**Step:** Step 4 of 90 — Windows Compatibility & Platform Support Policy  
**Status:** Approved Source Register Baseline  
**Date of Research:** 2026-11-08  

---

## 1. Register Overview & Evidence Taxonomy

This source register provides a fully auditable bibliographic record of primary sources, official Microsoft Learn documentation, Windows release datasheets, and toolchain specifications used to establish KinetiWall's Windows Platform Support Policy during Step 4.

### Evidence Confidence Classifications:
- **`[VERIFIED_FACT]`**: Directly extracted from official Microsoft Learn documentation, Windows Lifecycle datasheets, or Rust-lang target definitions.
- **`[EVIDENCE_SUPPORTED]`**: Substantiated by reproducible empirical benchmarks, open-source codebase audits, or maintainer issue trackers.
- **`[ENGINEERING_RECOMMENDATION]`**: A technically justified architectural choice based on comparative trade-off analysis.
- **`[PROVISIONAL]`**: A working assumption requiring empirical test execution during Phase 2.

---

## 2. Bibliographic Primary Source Records

### PSR-001: Windows 11 Version 24H2 Release Information & Lifecycle
- **Product / Feature:** Windows 11, Version 24H2 (OS Build 26100)
- **Source Title:** Microsoft Learn: Windows 11 Release Information & Lifecycle Datasheet
- **Direct Source URL:** https://learn.microsoft.com/en-us/windows/release-health/windows11-release-information
- **Publication / Access Date:** Continuous Microsoft Updates / Accessed: 2026-11-08
- **Evidence Classification:** `[VERIFIED_FACT]`
- **Evidence Established:** Windows 11 Version 24H2 reached General Availability on October 1, 2024, based on Build 26100. Consumer servicing runs through October 13, 2026; Enterprise servicing through October 12, 2027.
- **Implication for KinetiWall:** Must be certified as a Tier A Primary Supported operating system (`DEC-PLT-003`).

---

### PSR-002: Windows 10 Version 1903 Lifecycle & Servicing Baseline
- **Product / Feature:** Windows 10, Version 1903 (OS Build 18362)
- **Source Title:** Microsoft Learn: Windows 10 Release History & End of Servicing
- **Direct Source URL:** https://learn.microsoft.com/en-us/lifecycle/products/windows-10-home-and-pro
- **Publication / Access Date:** Official Lifecycle Archive / Accessed: 2026-11-08
- **Evidence Classification:** `[VERIFIED_FACT]`
- **Evidence Established:** Microsoft ended servicing for Windows 10 Version 1903 on December 8, 2020. Version 2004 (Build 19041) unified the servicing architecture for 2004 through 22H2.
- **Implication for KinetiWall:** Reconciles the user's original 1903 requirement by designating 1903 as Tier B (Supported with Limitations), while setting Build 19041+ as Tier A Primary (`DEC-PLT-001`).

---

### PSR-003: Windows 10 End of Support & Extended Security Updates (ESU)
- **Product / Feature:** Windows 10 Version 22H2 Retirement & ESU Timeline
- **Source Title:** Microsoft Lifecycle: Windows 10 22H2 Support Transition
- **Direct Source URL:** https://learn.microsoft.com/en-us/windows/whats-new/extended-security-updates
- **Publication / Access Date:** 2024–2026 / Accessed: 2026-11-08
- **Evidence Classification:** `[VERIFIED_FACT]`
- **Evidence Established:** Microsoft retired standard free support for Windows 10 on October 14, 2025. Consumer ESU extends critical security updates to October 12, 2027; Commercial ESU extends to October 10, 2028.
- **Implication for KinetiWall:** Guarantees Tier A support for Windows 10 22H2 through October 2027 to capture the massive active PC user base (`DEC-PLT-002`).

---

### PSR-004: Windows 11 24H2 Prism Emulation for ARM64 Devices
- **Product / Feature:** Prism Emulation Engine on Windows 11 on ARM
- **Source Title:** Microsoft Learn: How Windows on ARM Runs x86 and x64 Applications
- **Direct Source URL:** https://learn.microsoft.com/en-us/windows/arm/apps-on-arm-x86-emulation
- **Publication / Access Date:** 2024–2026 / Accessed: 2026-11-08
- **Evidence Classification:** `[VERIFIED_FACT]`
- **Evidence Established:** Windows 11 24H2 includes Prism, a high-efficiency binary translation engine supporting AVX and AVX2 vector instructions, enabling standard x64 Win32 applications to run smoothly on Snapdragon X hardware.
- **Implication for KinetiWall:** Allows x86-64 KinetiWall binaries to run on ARM64 Copilot+ laptops under Tier B (Supported with Limitations) out-of-the-box (`DEC-PLT-006`).

---

### PSR-005: Windows 11 24H2 Desktop Composition & Hidden Desktop Icons
- **Product / Feature:** Explorer Desktop Hierarchy & WorkerW Lifecycle on Build 26100
- **Source Title:** Windows Desktop Shell Issue Audits & Lively Wallpaper Issues #2500+
- **Direct Source URL:** https://github.com/rocksdanister/lively/issues
- **Publication / Access Date:** 2024–2026 / Accessed: 2026-11-08
- **Evidence Classification:** `[EVIDENCE_SUPPORTED]`
- **Evidence Established:** In Windows 11 24H2, if desktop icons are hidden, `explorer.exe` collapses the standard icon window hierarchy. Sending undocumented message `0x052C` causes `WorkerW` to detach or vanish, triggering infinite refresh loops or grey screens.
- **Implication for KinetiWall:** Mandates the implementation of KinetiWall's dual-mode shell attachment algorithm with direct `Progman` fallback (`KW-POL-003`, `TR-001`).

---

### PSR-006: Microsoft Edge WebView2 Distribution & Evergreen Preinstallation
- **Product / Feature:** WebView2 Runtime Deployment on Windows 10 & 11
- **Source Title:** Microsoft Learn: Distribute your app and the WebView2 Runtime
- **Direct Source URL:** https://learn.microsoft.com/en-us/microsoft-edge/webview2/concepts/distribution
- **Publication / Access Date:** Official Microsoft Documentation / Accessed: 2026-11-08
- **Evidence Classification:** `[VERIFIED_FACT]`
- **Evidence Established:** The Evergreen WebView2 Runtime is preinstalled on all Windows 11 and Windows 10 (Build 19041+) consumer and enterprise devices via standard updates. Fixed Version packages are available for air-gapped environments.
- **Implication for KinetiWall:** Supports the dual-installer strategy: standard installer (~56 MB) leveraging system Evergreen WebView2, and optional offline bundle (~236 MB) with Fixed Version WebView2 (`DEC-PLT-011`).

---

### PSR-007: Rust Target Support Tiers for Windows (`x86_64` vs `aarch64`)
- **Product / Feature:** Rust Platform Support Matrix
- **Source Title:** The rustc Book: Platform Support Tiers
- **Direct Source URL:** https://doc.rust-lang.org/nightly/rustc/platform-support.html
- **Publication / Access Date:** Continuous / Accessed: 2026-11-08
- **Evidence Classification:** `[VERIFIED_FACT]`
- **Evidence Established:** `x86_64-pc-windows-msvc` is a Tier 1 target with full automated CI guarantees. `aarch64-pc-windows-msvc` is a Tier 2 target guaranteed to build.
- **Implication for KinetiWall:** Reinforces x86-64 as the sole Tier A primary launch architecture, with native ARM64 designated as a Tier C roadmap spike (`DEC-PLT-004`, `DEC-PLT-005`).

---

### PSR-008: Direct3D 11 Feature Levels & Hardware Acceleration
- **Product / Feature:** Direct3D 11 Graphics Device Specification
- **Source Title:** Microsoft Learn: Direct3D 11 Feature Levels
- **Direct Source URL:** https://learn.microsoft.com/en-us/windows/win32/direct3d11/overviews-direct3d-11-devices-downlevel-intro
- **Publication / Access Date:** Official Documentation / Accessed: 2026-11-08
- **Evidence Classification:** `[VERIFIED_FACT]`
- **Evidence Established:** Feature Level 10_0 supports basic D3D11 rendering on legacy Intel HD Graphics 4000. Feature Level 11_1 is required for flip-model swapchains (`DXGI_SWAP_EFFECT_FLIP_DISCARD`).
- **Implication for KinetiWall:** Establishes `D3D_FEATURE_LEVEL_10_0` as the absolute functional minimum, and `11_1` as recommended (`DEC-PLT-008`).

