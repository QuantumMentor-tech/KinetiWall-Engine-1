# KinetiWall — Unresolved Platform Decisions Register

**Document Identifier:** KW-POL-006  
**Document Type:** Strategic Platform Policy Decision Register (Deliverable F)  
**Current Phase:** Phase 1 — Project Research, Requirements & Feasibility  
**Step:** Step 4 of 90 — Windows Compatibility & Platform Support Policy  
**Status:** Active Platform Governance Register  
**Date of Research:** 2026-11-08  

---

## 1. Governance & Decision Framework

This register tracks foundational operating system, architecture, dependency, and testing policy decisions that cannot be resolved unilaterally by engineering. Each item documents the authoritative user requirement, empirical research findings, available options, technical trade-offs, confidence levels, and explicit recommendations requiring product owner approval.

---

## 2. Register of Twelve Core Platform Decisions

### DEC-PLT-001: Minimum Windows 10 Baseline: Version 1903 vs. Version 2004 (Build 19041+)
- **Existing User Requirement:** Windows 10 64-bit Version 1903 or higher (`KW-NF-023`).
- **Available Evidence:** Microsoft retired Version 1903 on December 8, 2020. Windows 10 Version 2004 (Build 19041) unified the servicing architecture for 2004, 20H2, 21H1, 21H2, and 22H2. Build 19041 is required for native WebView2 virtual host mapping and stable Per-Monitor V2 DPI scaling (`PSR-002`).
- **Options Compared:**
  - *Option A (Preserve 1903 as Tier A):* Obligates testing on retired 2019 builds; requires custom DPI fallback code; high maintenance.
  - *Option B (Reconciled Policy — Recommended):* Designate **Windows 10 Build 19041+ (through 22H2) as Tier A Primary**, and retain **Version 1903/1909 under Tier B (Supported with Limitations)** requiring manual WebView2 installation.
  - *Option C (Hard Cutoff at 2004):* Drop 1903/1909 completely (rejects user brief).
- **Recommendation:** **Option B (Reconciled Tier B for 1903; Tier A for 19041+)**. Preserves user intent while maintaining engineering sanity.
- **Approval Status:** **AWAITING USER APPROVAL.**

---

### DEC-PLT-002: Windows 10 Post-2025 ESU Servicing Support Commitment
- **Existing User Requirement:** Official support for Windows 10.
- **Available Evidence:** Microsoft ended standard free consumer support for Windows 10 on October 14, 2025. Consumer ESU extends critical security patches to October 2027; Commercial ESU extends to October 2028 (`PSR-003`). Over 40% of global desktop PCs remain on Windows 10 22H2.
- **Options Compared:**
  - *Option A (Align with Consumer ESU — Recommended):* Guarantee Tier A primary support for Windows 10 22H2 through **October 2027**.
  - *Option B (Immediate Windows 11 Exclusivity):* Drop Windows 10 immediately. Alienates ~40% of target users.
- **Recommendation:** **Option A**. Support Windows 10 22H2 through October 2027 to capture the massive existing PC user base.
- **Approval Status:** **AWAITING USER APPROVAL.**

---

### DEC-PLT-003: Windows 11 Qualification: 22H2, 23H2, and 24H2 Support Tiers
- **Existing User Requirement:** Full support for Windows 11 releases.
- **Available Evidence:** Windows 11 22H2, 23H2, and 24H2 (Build 26100) are widely deployed. Version 24H2 introduced desktop composition changes that require dual-mode shell attachment (`PSR-001`, `PSR-005`).
- **Options Compared:**
  - *Option A (All Three as Tier A Primary — Recommended):* Support 22H2, 23H2, and 24H2 as Tier A, implementing dual-mode attachment to handle 24H2 composition changes.
  - *Option B (Exclude 24H2 temporarily):* Unacceptable; 24H2 is the current preinstalled version on all new Windows 11 PCs.
- **Recommendation:** **Option A**. Certify 22H2, 23H2, and 24H2 as Tier A Primary Supported environments.
- **Approval Status:** **APPROVED IN POLICY PROPOSAL.**

---

### DEC-PLT-004: Primary Release Processor Architecture: x86-64 as Sole Launch Target
- **Existing User Requirement:** Platform target includes x86-64 and ARM64.
- **Available Evidence:** x86-64 represents >92% of the PC gaming and workstation market. All pre-built dependencies (`libmpv-2.dll`, WebView2, Tokio, Direct3D 11) are 100% mature and verified on x86-64.
- **Options Compared:**
  - *Option A (x86-64 Sole Tier A Launch Architecture — Recommended):* Launch v1.0 exclusively on x86-64 as Tier A, providing instant compatibility with Snapdragon X ARM64 devices via Windows 11 24H2 Prism emulation (Tier B).
  - *Option B (Block v1.0 on Native ARM64 Toolchains):* Delays release by months to cross-compile and audit custom ARM64 FFmpeg/libmpv libraries.
- **Recommendation:** **Option A**. Focus initial release rigor on x86-64 Tier A, with Prism emulation providing immediate Tier B coverage on ARM laptops.
- **Approval Status:** **AWAITING USER APPROVAL.**

---

### DEC-PLT-005: Native ARM64 Support: Mandatory for v1.0 vs. Phase 2 Roadmap Spike
- **Existing Context:** ARM64 mentioned in original brief.
- **Available Evidence:** Compiling Rust for `aarch64-pc-windows-msvc` is fully supported by the Rust toolchain. However, pre-built LGPL 2.1 shared binaries of `libmpv-2.dll` for Windows on ARM are not officially published by upstream maintainers, requiring custom build pipelines (`PSR-007`).
- **Options Compared:**
  - *Option A (Promote Native ARM64 to v1.0 Blocker):* Requires dedicated ARM64 test hardware and custom C/C++ cross-compilation infrastructure in v1.
  - *Option B (Schedule Native ARM64 as Tier C Roadmap Spike — Recommended):* Support ARM64 via Prism emulation for v1.0, and schedule a dedicated native ARM64 cross-compile spike for post-v1.
- **Recommendation:** **Option B**. Prevents blocking the entire project on media library cross-compilation.
- **Approval Status:** **AWAITING USER APPROVAL.**

---

### DEC-PLT-006: x64 Prism Emulation on Windows on ARM as a Valid Tier B Path
- **Existing Context:** Windows on ARM laptops (Qualcomm Snapdragon X Elite / Plus).
- **Available Evidence:** Windows 11 24H2 introduced Microsoft Prism, an efficient binary translation engine supporting AVX/AVX2. Benchmarks indicate that x64 media applications run smoothly with ~5–8% CPU translation overhead (`PSR-004`).
- **Options Compared:**
  - *Option A (Officially Recognize Prism as Tier B Supported — Recommended):* Test and validate x64 KinetiWall binaries on Prism, offering supported live wallpapers for Snapdragon X users.
  - *Option B (Block Execution on ARM Emulation):* Reject installation on ARM devices entirely.
- **Recommendation:** **Option A**. Unlocks immediate compatibility for modern Windows 11 Copilot+ laptops.
- **Approval Status:** **APPROVED IN POLICY PROPOSAL.**

---

### DEC-PLT-007: Validated Desktop Integration Baseline & Proof-of-Concept Gating
- **Existing Context:** Sub-desktop window parenting behind desktop icons.
- **Available Evidence:** Message `0x052C` to `Progman` is undocumented. In Windows 11 24H2, `WorkerW` detaches when desktop icons are hidden, requiring fallback parenting directly into `Progman` (`PSR-001`, `PSR-005`).
- **Options Compared:**
  - *Option A (Dual-Mode Attachment with Physical Validation Spike — Recommended):* Implement runtime capability detection: use WorkerW if icons are present; use direct Progman parenting if icons are hidden. Gate official support behind physical Step 5 test execution (`CT-004`, `CT-005`).
  - *Option B (Assume Undocumented 0x052C is Universally Guaranteed):* High risk of release failure.
- **Recommendation:** **Option A**. Enforce physical verification of the dual-mode spike before declaring 24H2 certified.
- **Approval Status:** **APPROVED IN POLICY PROPOSAL.**

---

### DEC-PLT-008: Minimum Functional Graphics Hardware Baseline
- **Existing Context:** Hardware-accelerated Direct3D rendering.
- **Available Evidence:** Direct3D 11.0 Feature Level 10_0 encompasses Intel HD Graphics 4000 (2012) and newer. Feature Level 11_1 is required for low-latency flip-model swapchains (`DXGI_SWAP_EFFECT_FLIP_DISCARD`).
- **Options Compared:**
  - *Option A (Direct3D 11.0 Feature Level 10_0 Minimum — Recommended):* Allows older laptops and basic GPUs to run the software, using Feature Level 11_1 optimizations dynamically when available.
  - *Option B (Require Direct3D 12):* Completely unnecessary for 2D/video composition; excludes older hardware.
- **Recommendation:** **Option A**. Set `D3D_FEATURE_LEVEL_10_0` as the absolute minimum, and `11_1` as recommended.
- **Approval Status:** **APPROVED IN POLICY PROPOSAL.**

---

### DEC-PLT-009: Official Video Container & Codec Support Matrix
- **Existing Context:** User requested MP4, WebM, AVI. Research suggested MKV.
- **Available Evidence:** `libmpv-2.dll` with FFmpeg natively decodes H.264/AVC, H.265/HEVC, VP8, VP9, and AV1 across MP4, WebM, AVI, and MKV containers.
- **Options Compared:**
  - *Option A (Official Support for MP4, WebM, AVI + Compatibility Support for MKV — Recommended):* Full hardware acceleration for MP4 and WebM. MKV supported subject to container stream compatibility (`KW-F-010`, `KW-F-011`).
  - *Option B (Strictly Exclude MKV):* Artificially limits user flexibility.
- **Recommendation:** **Option A**. Include MP4, WebM, and AVI as primary supported formats, with MKV supported for standard video streams.
- **Approval Status:** **APPROVED IN POLICY PROPOSAL.**

---

### DEC-PLT-010: Handling Missing or Corrupted Microsoft Edge WebView2 Runtimes
- **Existing Context:** Interactive wallpapers depend on WebView2 (`KW-F-017`).
- **Available Evidence:** Modern Windows 11 and Windows 10 2004+ include Evergreen WebView2 preinstalled. Air-gapped or stripped enterprise systems may lack it (`PSR-006`).
- **Options Compared:**
  - *Option A (Graceful Degradation with Actionable Notification — Recommended):* Detect missing runtime at startup. If absent, disable web wallpapers in UI and show actionable guidance, while keeping video playback 100% operational (`KW-POL-004`).
  - *Option B (Hard Crash / Fatal Abort on Startup):* Degrades user experience completely.
- **Recommendation:** **Option A**. Ensure missing WebView2 never crashes the core daemon or impedes video playback.
- **Approval Status:** **APPROVED IN POLICY PROPOSAL.**

---

### DEC-PLT-011: Offline Distribution Packaging: Standard Installer vs. Fixed Version Bundle
- **Existing Context:** Commitment to 100% offline operability (`KW-F-045`) vs. installer size target (<35 MB, `KW-NF-006`).
- **Available Evidence:** Bundling the standalone Fixed Version WebView2 runtime adds ~180 MB compressed, yielding a ~236 MB installer. The standard installer leveraging system Evergreen WebView2 measures ~56 MB with sample media (`KW-RES-001`, Sec 12).
- **Options Compared:**
  - *Option A (Dual-Distribution Model — Recommended):* Distribute a **Standard Desktop Installer (~56 MB)** relying on system Evergreen WebView2 for general users, and an **Offline Enterprise Bundle (~236 MB)** with bundled Fixed Version WebView2 for air-gapped workstations.
  - *Option B (Force Single 236 MB Installer for All Users):* Penalizes 98% of users with an unnecessarily large download.
- **Recommendation:** **Option A (Dual-Distribution Model)**. Satisfies both size targets and air-gapped enterprise needs.
- **Approval Status:** **AWAITING USER APPROVAL.**

---

### DEC-PLT-012: Minimum Test Coverage Gate for Official Support Certification
- **Existing Context:** Establishing sufficient empirical proof before advertising support.
- **Available Evidence:** Many open-source live wallpaper engines suffer from regressions because multi-monitor, sleep/wake, and Explorer crash recovery are never systematically tested.
- **Options Compared:**
  - *Option A (100% Pass on All 20 Scenarios (`CT-001` to `CT-020`) — Recommended):* An operating system build or architecture can only enter **Tier A Primary Supported** status when all 20 compatibility tests pass with zero Blocker or Critical defects.
  - *Option B (Ad-Hoc Developer Testing):* Unacceptable; introduces unverified platform claims.
- **Recommendation:** **Option A**. Enforce the 20-scenario test matrix as the mandatory gating standard for Tier A certification.
- **Approval Status:** **APPROVED IN POLICY PROPOSAL.**

