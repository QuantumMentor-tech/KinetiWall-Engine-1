# KinetiWall — Prioritized Opportunity Register

**Document Identifier:** KW-COMP-006  
**Document Type:** Strategic Opportunity Prioritization & Roadmap Gating (Deliverable F)  
**Current Phase:** Phase 1 — Project Research, Requirements & Feasibility  
**Step:** Step 3 of 90 — Competitor Analysis & Product Differentiation  
**Status:** Approved Opportunity Register  
**Date of Research:** 2026-11-08  

---

## 1. Prioritization Framework & Methodology

To ensure KinetiWall maintains strict scope discipline and avoids feature creep while capitalizing on competitor weaknesses, all product opportunities are evaluated using a multi-dimensional qualitative assessment model:

### Evaluation Dimensions:
1. **User Value (UV):** Degree of direct utility or pain relief for target users (High, Medium, Low).
2. **Evidence of Unmet Need (EUN):** Strength of empirical evidence that existing competitors fail to solve this problem (Verified, Strong, Moderate).
3. **Requirement Alignment (RA):** Relationship to the Step 1 Approved Requirements Baseline (Directly Aligned, Extension, New Feature).
4. **Technical Feasibility (TF):** Proven engineering path in Rust/Win32 without speculative dependencies (High, Medium, Complex).
5. **Implementation Cost (IC):** Engineering effort required across the 90-step roadmap (Low, Moderate, High).
6. **Performance & Security Impact (PSI):** Effect on memory, CPU, battery, and sandbox integrity (Positive, Neutral, Hazard).
7. **Differentiation Potential (DP):** Ability to distinguish KinetiWall in market comparisons (High, Moderate, Table Stakes).

### Scope Categorization:
- **Category A (Required Baseline Capabilities to Execute Flawlessly):** Already approved in Step 1. These must be executed with extreme engineering rigor.
- **Category B (Potential Improvements Requiring Explicit User Review):** Research-derived enhancements that offer strong value but require formal approval before inclusion.
- **Category C (Future Product Opportunities):** High-effort or community-dependent features intentionally deferred to post-v1 milestones.

---

## 2. Category A: Required Baseline Capabilities (Execute Flawlessly)

These capabilities are already confirmed in the Step 1 Requirements Inventory. Competitor research emphasizes that KinetiWall must execute these with superior stability and performance compared to existing tools.

| ID | Title & Focus | UV | EUN | RA | TF | IC | PSI | DP | Key Competitive Imperative |
| :--- | :--- | :---: | :---: | :---: | :---: | :---: | :---: | :---: | :--- |
| **OPP-A01** | **Sub-10 MB Idle Daemon & Ephemeral UI** (`KW-F-007`, `KW-NG-001`) | **High** | Verified | Direct | High | Mod | Positive | **High** | Eliminates the 150–250 MB memory bloat seen in Lively Wallpaper. |
| **OPP-A02** | **Windows 11 24H2 Dual-Mode Shell Attachment** (`KW-F-001`, `UDR-002`) | **High** | Verified | Direct | High | Mod | Neutral | **High** | Solves the recurring 24H2 screen flashing and WorkerW crashes. |
| **OPP-A03** | **Hardware-Accelerated Gapless Video Looping** (`KW-F-012`, `KW-F-013`) | **High** | Strong | Direct | High | Mod | Positive | Table Stakes | Must achieve sub-16.6ms loop jitter via `libmpv` with `vo=gpu-next`. |
| **OPP-A04** | **Sub-100ms Event-Driven Fullscreen Pause** (`KW-F-025`, `KW-F-028`) | **High** | Strong | Direct | High | Low | Positive | Moderate | Event-driven hooks eliminate polling wakeups and protect 1% low FPS. |
| **OPP-A05** | **100% Offline Local Operation & Zero Accounts** (`KW-F-045`, `KW-NF-019`) | **High** | Verified | Direct | High | Low | Positive | **High** | Standalone alternative to Steam-locked tools; zero telemetry. |
| **OPP-A06** | **Edge WebView2 Sandboxed Virtual Host** (`KW-F-017`, `KW-F-018`) | **High** | Strong | Direct | High | Mod | Positive | Moderate | Isolates web wallpapers from local files via `https://kinetiwall.local/`. |

---

## 3. Category B: Potential Improvements (Awaiting User Approval)

These research-backed enhancements emerged from competitor analysis as valuable quality-of-life additions. They do not alter the core v1 architecture but offer high ROI if approved for inclusion in early release milestones.

### OPP-B01: Three-Tier Configurable Deactivation States (Pause vs Suspend vs Stop)
- **Concept:** Provide a user toggle: "Gaming Mode: Instant Resume (Pause decoding, retain VRAM)" vs "VRAM Conservation (Stop player, release all GPU surfaces)".
- **Justification:** Hardcore gamers running 4K/VR titles on 8GB VRAM cards desperately need all VRAM freed, whereas general users prefer <30ms resume latency.
- **Evaluation:** UV: High | EUN: Moderate | RA: Extension (`KW-F-026`, `UDR-008`) | TF: High | IC: Low | PSI: Positive | DP: High.
- **Recommendation:** **Approve for v1.0 configuration settings.**

---

### OPP-B02: Global Hotkey Quick-Switcher HUD Overlay
- **Concept:** Lightweight transparent overlay triggered via `Win+Ctrl+W` to switch wallpapers, mute audio, or pause rendering without opening the full gallery window (`OFF-002`).
- **Justification:** Avoids launching the heavier gallery process for simple day-to-day interactions.
- **Evaluation:** UV: Medium | EUN: Moderate | RA: Extension (`OFF-002`) | TF: High | IC: Low | PSI: Neutral | DP: Moderate.
- **Recommendation:** **Schedule for v1.1 milestone.**

---

### OPP-B03: Portable Wallpaper Package Archive (`.kwpack`)
- **Concept:** Standardize a zip-based container format containing video/web assets, thumbnail, and `kinetiwall.json` manifest for drag-and-drop import (`OFF-007`).
- **Justification:** Allows users to easily backup, share, and import wallpapers without Steam Workshop.
- **Evaluation:** UV: High | EUN: Strong | RA: Aligned (`KW-F-034`) | TF: High | IC: Low | PSI: Neutral | DP: Moderate.
- **Recommendation:** **Incorporate into v1.0 asset packaging specification.**

---

## 4. Category C: Future Product Opportunities (Explicitly Deferred)

These opportunities represent high-effort initiatives, third-party ecosystem dependencies, or massive maintenance burdens that must **not** be admitted into the initial release baseline.

| Opportunity ID | Proposed Capability | Technical Dependencies & Risks | Strategic Recommendation |
| :--- | :--- | :--- | :--- |
| **OPP-C01** | **In-App 2D/3D Scene Animation Editor** | Massive UI & timeline engine; multi-year engineering commitment; high complexity. | **Strictly Exclude from v1 & v2.** Rely on external authoring tools (Blender, WebGL, Handbrake). |
| **OPP-C02** | **Centralized Online Community Workshop & CDN** | Backend server infrastructure, CDN bandwidth costs, content moderation, copyright liabilities. | **Deferred to v2.0+;** evaluate peer-to-peer or decentralized catalog options. |
| **OPP-C03** | **Standalone 3D / Game Engine Executable Hosting** | Severe anti-cheat false positives, handle leaks, security hazards (`KW-F-044`, `KW-NG-006`). | **Deferred / Research Only;** satisfy 3D needs safely via WebGL2 in web player. |
| **OPP-C04** | **Hardware RGB Peripheral Synchronization** | Proprietary vendor SDKs (Razer Synapse, Corsair iCUE) are notoriously bloated and unstable. | **Deferred to Post-v1 Plugin Architecture** (evaluate lightweight OpenRGB integration). |
| **OPP-C05** | **Local AI Video Upscaling & Frame Interpolation** | 2.5–5.0 GB VRAM consumption, GPU contention during gaming (`OFF-012`, `UDR-012`). | **Strictly Excluded from Core Engine;** evaluate as an independent offline CLI companion tool. |

