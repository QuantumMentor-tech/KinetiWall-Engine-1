# KINETIWALL — DEPENDENCY SELECTION, LICENSING & SUPPLY-CHAIN POLICY
## Document ID: KW-DEP-000 | Phase 2: System Architecture & Technical Decisions

---

### Executive Summary

The **KinetiWall Dependency Selection, Licensing & Supply-Chain Governance Policy** establishes the authoritative framework for evaluating, approving, managing, securing, and distributing third-party software components across the KinetiWall ecosystem.

KinetiWall combines native Rust systems engineering, Windows Win32 API integration, multi-process hardware-accelerated media playback (mpv/libmpv), sandboxed web runtimes (Microsoft Edge WebView2), and an on-demand desktop user interface (Tauri v2 + React). To maintain KinetiWall's core architectural tenets—**local-first operation, < 100 MB memory footprint, < 800ms startup, zero malware/telemetry, and clean license compliance**—this policy establishes:
1. **Minimum-Dependency Principle**: Code is evaluated against standard library and native OS capabilities before introducing external crates or npm packages.
2. **Deterministic Licensing Compliance**: Absolute transparency and separation of permissive licenses (MIT, Apache-2.0, BSD-3-Clause) from copyleft components (LGPL-2.1+, GPL-2.0/3.0). Explicit build-configuration controls govern libmpv/FFmpeg distribution to avoid unintentional license contamination.
3. **Hardened Supply-Chain Security**: Mandatory cryptographic lockfile pinning (`Cargo.lock`, `pnpm-lock.yaml`), automated vulnerability auditing (`cargo-audit`, GitHub Advisory Database), and provenance verification for all native binary redistributables.
4. **Standardized SBOM and Attribution**: Automated CycloneDX JSON Software Bill of Materials (SBOM) generation and automated license notice assembly (`cargo-about`) bundled into every release.
5. **Offline Installation Architecture**: Strict classification of core offline capabilities versus optional online components, ensuring KinetiWall functions completely without Internet connectivity.

---

### Document Navigation Index

| Document ID | File Name | Scope & Key Contents |
| :--- | :--- | :--- |
| **KW-DEP-000** | [README.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/dependency_and_licensing/README.md) | Policy overview, classification framework, navigation index, and requirements traceability matrix. |
| **KW-DEP-001** | [DEPENDENCY_INVENTORY.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/dependency_and_licensing/DEPENDENCY_INVENTORY.md) | Exhaustive inventory of 38 candidate dependencies across 11 functional subsystems with 24 auditable attributes. |
| **KW-DEP-002** | [SELECTION_AND_EVALUATION_POLICY.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/dependency_and_licensing/SELECTION_AND_EVALUATION_POLICY.md) | Dependency evaluation criteria, minimum-dependency framework, approval workflow, deprecation, and removal protocols. |
| **KW-DEP-003** | [LICENSE_COMPLIANCE_AND_REDISTRIBUTION.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/dependency_and_licensing/LICENSE_COMPLIANCE_AND_REDISTRIBUTION.md) | SPDX license mapping, copyleft obligations, libmpv LGPL vs GPL build flags, patent review, binary distribution register, trademark policy. |
| **KW-DEP-004** | [SUPPLY_CHAIN_SECURITY_AND_LOCKFILES.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/dependency_and_licensing/SUPPLY_CHAIN_SECURITY_AND_LOCKFILES.md) | Package provenance, cryptographic lockfiles, hash verification, typosquatting defense, build-script sandboxing, CI release checks. |
| **KW-DEP-005** | [SBOM_AND_NOTICES_POLICY.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/dependency_and_licensing/SBOM_AND_NOTICES_POLICY.md) | CycloneDX JSON SBOM generation specification, automated attribution notice generation (`cargo-about`), distribution requirements. |
| **KW-DEP-006** | [VULNERABILITY_RESPONSE_AND_MAINTENANCE.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/dependency_and_licensing/VULNERABILITY_RESPONSE_AND_MAINTENANCE.md) | Vulnerability triage matrix, CVSS scoring, SLA response windows, automated audit pipelines, emergency patching protocol. |
| **KW-DEP-007** | [OFFLINE_DISTRIBUTION_AND_DECISIONS.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/dependency_and_licensing/OFFLINE_DISTRIBUTION_AND_DECISIONS.md) | Offline deployment packaging, installer footprint budgets, Dependency Decision Register (`DEC-DEP-01`..`08`), and risk register. |

---

### Dependency Classification Framework

Dependencies are classified according to two orthogonal dimensions: **Approval Status** and **Functional Subsystem**.

#### Approval Status Taxonomy
- **EXISTING**: Present in the repository or build configuration (currently zero, as the repository is in design/pre-implementation phase).
- **APPROVED**: Formally accepted by architectural decisions (`ADR-0001` through `ADR-0014`, `DEC-SEC-01` through `DEC-SEC-08`, `DEC-DATA-01` through `DEC-DATA-06`).
- **PROPOSED**: Evaluated candidate recommended for implementation baseline.
- **ALTERNATIVE**: Viable candidate evaluated but not currently recommended.
- **OPTIONAL**: Component needed only for non-mandatory features (e.g. advanced audio visualization, optional shaders).
- **TRANSITIVE**: Ingested indirectly through direct dependencies.
- **DEPRECATED**: Phased out or scheduled for replacement.
- **REJECTED**: Formally evaluated and rejected with recorded technical rationale (e.g., pure SQLite RDBMS in V1.0).
- **UNKNOWN**: Insufficient data to determine feasibility or license status.

---

### Requirements Traceability Matrix

| Requirement ID | Requirement Description | Governing Dependency & Policy Document |
| :--- | :--- | :--- |
| **KW-F-001** | Multi-process native Rust controller | [DEPENDENCY_INVENTORY.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/dependency_and_licensing/DEPENDENCY_INVENTORY.md) (`DEP-01`: `windows-rs`, `DEP-02`: `tokio`) |
| **KW-F-006** | Native Win32 desktop integration (`WorkerW`) | [DEPENDENCY_INVENTORY.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/dependency_and_licensing/DEPENDENCY_INVENTORY.md) (`DEP-01`: `windows-rs::Win32::UI::WindowsAndMessaging`) |
| **KW-F-015** | Video wallpaper playback engine (MP4, MKV, WebM) | [LICENSE_COMPLIANCE_AND_REDISTRIBUTION.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/dependency_and_licensing/LICENSE_COMPLIANCE_AND_REDISTRIBUTION.md) (`DEP-07`: `libmpv` LGPL build) |
| **KW-F-016** | Web wallpaper execution & sandboxing | [LICENSE_COMPLIANCE_AND_REDISTRIBUTION.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/dependency_and_licensing/LICENSE_COMPLIANCE_AND_REDISTRIBUTION.md) (`DEP-09`: Microsoft WebView2 Evergreen) |
| **KW-F-022** | Thumbnail generation & image caching | [DEPENDENCY_INVENTORY.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/dependency_and_licensing/DEPENDENCY_INVENTORY.md) (`DEP-15`: `image` / `webp` pure Rust) |
| **KW-F-024** | WASAPI audio visualization & FFT | [DEPENDENCY_INVENTORY.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/dependency_and_licensing/DEPENDENCY_INVENTORY.md) (`DEP-13`: `cpal`, `DEP-14`: `rustfft`) |
| **KW-F-038** | On-demand settings and gallery interface | [DEPENDENCY_INVENTORY.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/dependency_and_licensing/DEPENDENCY_INVENTORY.md) (`DEP-10`: `tauri` v2, `DEP-11`: React/Vite) |
| **KW-NF-001**| Memory footprint < 100 MB | [SELECTION_AND_EVALUATION_POLICY.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/dependency_and_licensing/SELECTION_AND_EVALUATION_POLICY.md) §2 (Zero C FFI bloat, pure Rust) |
| **KW-NF-006**| Cold startup latency < 800ms | [SELECTION_AND_EVALUATION_POLICY.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/dependency_and_licensing/SELECTION_AND_EVALUATION_POLICY.md) §2 (Fast zero-copy `serde_json`) |
| **KW-NF-018**| Sandboxed execution of untrusted content | [LICENSE_COMPLIANCE_AND_REDISTRIBUTION.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/dependency_and_licensing/LICENSE_COMPLIANCE_AND_REDISTRIBUTION.md) (`DEP-09`: WebView2 Sandboxing) |
| **KW-NG-002**| Non-Goal: Third-party native `.exe` wallpapers | [OFFLINE_DISTRIBUTION_AND_DECISIONS.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/dependency_and_licensing/OFFLINE_DISTRIBUTION_AND_DECISIONS.md) (`DEC-DEP-06`: Reject external binary execution) |
| **ADR-0009** | Hybrid Configuration Architecture (No SQLite V1) | [OFFLINE_DISTRIBUTION_AND_DECISIONS.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/dependency_and_licensing/OFFLINE_DISTRIBUTION_AND_DECISIONS.md) (`DEC-DEP-05`: SQLite deferred) |

