# KINETIWALL — DATA ARCHITECTURE & STORAGE STRATEGY
## Document ID: KW-DAT-000 | Phase 2: System Architecture & Technical Decisions

---

### Executive Summary

The **KinetiWall Data Architecture Specification** establishes the authoritative blueprint for data domains, entity relationships, physical storage strategies, configuration schemas, asset lifecycle management, migration protocols, and data security across the KinetiWall engine.

In alignment with **ADR-0009 (Storage & Configuration Architecture)** and the multi-process boundary architecture defined in **Step 8 (KW-PRC-001..005)**, KinetiWall adopts a **Deterministic Hybrid Storage Architecture**:
1. **Application Settings & Active Monitor Topology**: Stored as a single, human-readable, schema-versioned JSON document (`config.json`) manipulated strictly via atomic temp-file replace patterns with automatic `.bak` fallback.
2. **Wallpaper Package Storage & Manifest Declarations**: Decentralized folder-per-wallpaper structure under `%LOCALAPPDATA%\KinetiWall\wallpapers\{wallpaper_id}\`, each governed by a standardized, self-contained `project.json` (Manifest Schema V1).
3. **Media & Web Assets**: Raw filesystem storage partitioned by package identity, separating engine-managed library assets from external user-linked media.
4. **Reproducible Cache**: Transient WebP/PNG thumbnails and GPU pipeline shader caches stored in dedicated cache directories with deterministic LRU eviction limits.
5. **Single-Writer Guarantee**: The Controller Core (`kinetiwall-core.exe`, specifically `ConfigurationRepository` and `LibraryRepository`) is the **sole authorized writer** to persistent configuration and library storage. Player processes and UI frontends access data strictly through read-only memory projections or IPC contracts.

---

### Document Navigation Index

| Document ID | File Name | Scope & Key Contents |
| :--- | :--- | :--- |
| **KW-DAT-000** | [README.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/data_architecture/README.md) | Architectural index, core principles, data governance, and requirements traceability matrix. |
| **KW-DAT-001** | [DATA_DOMAINS_AND_OWNERSHIP.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/data_architecture/DATA_DOMAINS_AND_OWNERSHIP.md) | Comprehensive inventory of 9 persistent and transient data domains, ownership boundaries, access tiers, and lifecycle policies. |
| **KW-DAT-002** | [CONCEPTUAL_DATA_MODEL.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/data_architecture/CONCEPTUAL_DATA_MODEL.md) | Formal Entity-Relationship model, Mermaid ER diagram, primary identification strategy (UUIDv4 vs Hash), cascade rules, and domain invariants. |
| **KW-DAT-003** | [STORAGE_TECHNOLOGY_COMPARISON.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/data_architecture/STORAGE_TECHNOLOGY_COMPARISON.md) | Comparative evaluation of SQLite (`rusqlite`), Pure JSON, Hybrid, and TOML across 12 criteria, justifying the selected Hybrid Architecture. |
| **KW-DAT-004** | [CONFIGURATION_AND_PROPERTY_SCHEMAS.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/data_architecture/CONFIGURATION_AND_PROPERTY_SCHEMAS.md) | Exhaustive JSON schemas for `config.json` and `project.json` (Manifest V1), dynamic property types, validation rules, and override precedence. |
| **KW-DAT-005** | [FILE_STORAGE_AND_ASSET_LIFECYCLE.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/data_architecture/FILE_STORAGE_AND_ASSET_LIFECYCLE.md) | Canonical Windows directory layout (`%LOCALAPPDATA%` and Portable mode), staging pipelines, thumbnail lifecycle, reference counting, and cleanup algorithms. |
| **KW-DAT-006** | [SCHEMA_VERSIONING_AND_MIGRATIONS.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/data_architecture/SCHEMA_VERSIONING_AND_MIGRATIONS.md) | Independent domain versioning, forward/backward compatibility, sequential step-migration pipeline, and interrupted-write rollback recovery. |
| **KW-DAT-007** | [INTEGRITY_SECURITY_AND_DECISIONS.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/data_architecture/INTEGRITY_SECURITY_AND_DECISIONS.md) | Atomic write guarantees, Zip Slip mitigation, decompression bomb defense, privacy rules, backup/export protocols, and Open Architecture Decisions (`DEC-DATA-01`..`06`). |

---

### Core Data Governance Principles

1. **Zero Uncontrolled Mutation (Single Authoritative Writer)**:
   Only the Core Engine process (`kinetiwall-core.exe`) writes to persistent application and library state. Frontends (`kinetiwall-ui.exe`) and Wallpaper Players (`kinetiwall-player-video.exe`, `kinetiwall-player-web.exe`) hold ephemeral copies or receive snapshot updates over authenticated IPC (`IPC-01` through `IPC-03`). Direct disk mutation by child processes is strictly forbidden.
2. **Crash-Resistant Atomic Commit**:
   All state updates to configuration files use atomic temporary file creation followed by Windows file replacement semantics (`MoveFileExW(MOVEFILE_REPLACE_EXISTING)`). An interrupted write or power failure leaves the previous valid state or its backup intact.
3. **Decentralized Package Portability**:
   Wallpaper packages are self-describing, directory-contained entities. A package folder can be copied, backed up, or transferred to another machine with its assets and manifest intact, requiring no monolithic database export.
4. **Defense-in-Depth Import Validation**:
   Imported archives (`.kwp`, `.zip`) are treated as untrusted bytecode. Path traversals (Zip Slip), absolute paths, symbolic links, archive decompression bombs, and executable payloads (`.exe`, `.dll`, `.bat`, `.ps1`) are neutralized prior to extraction.
5. **Strict Separation of User Media and Managed Cache**:
   Original user video files are never modified, re-encoded in-place, or deleted without explicit user confirmation. Cached thumbnails, extracted temporary files, and shader compilations are treated as fully disposable and automatically reconstructible.
6. **Zero Cloud PII & Local Privacy**:
   All telemetry, crash logs, and operational metrics are strictly local, anonymized, and subject to automatic retention eviction (max 7 days). No wallpaper usage history or personal data is transmitted off the local machine.

---

### Requirements Traceability Matrix

The data architecture directly addresses and satisfies requirements defined in **Step 1 (KW-F-xxx, KW-NF-xxx)**, **Step 7 (ADR-xxx)**, and **Step 8 (CTR-xxx)**:

| Requirement ID | Requirement Description | Data Architecture Deliverable & Component |
| :--- | :--- | :--- |
| **KW-F-015** | Video wallpaper import (MP4, MKV, WebM) | `FILE_STORAGE_AND_ASSET_LIFECYCLE.md` §2.2 (Managed Asset Store) |
| **KW-F-016** | Web wallpaper package import (`index.html` + assets) | `CONFIGURATION_AND_PROPERTY_SCHEMAS.md` §3 (Manifest V1), `INTEGRITY_SECURITY_AND_DECISIONS.md` §2 |
| **KW-F-018** | Custom dynamic wallpaper properties (sliders, colors) | `CONFIGURATION_AND_PROPERTY_SCHEMAS.md` §4 (Property Definition & Override Schemas) |
| **KW-F-020** | Multi-monitor independent wallpaper assignment | `CONCEPTUAL_DATA_MODEL.md` §3 (MonitorAssignment Entity), `CONFIGURATION_AND_PROPERTY_SCHEMAS.md` §2 |
| **KW-F-022** | Thumbnail and preview generation & display | `DATA_DOMAINS_AND_OWNERSHIP.md` §2.7, `FILE_STORAGE_AND_ASSET_LIFECYCLE.md` §3 (Cache Eviction) |
| **KW-F-030** | Autostart on Windows logon | `CONFIGURATION_AND_PROPERTY_SCHEMAS.md` §2 (`general.autostart` setting) |
| **KW-F-035** | Audio mute & volume control per monitor/wallpaper | `CONFIGURATION_AND_PROPERTY_SCHEMAS.md` §2 (`audio` settings & monitor property overrides) |
| **KW-F-040** | Export & backup of custom wallpaper packages | `INTEGRITY_SECURITY_AND_DECISIONS.md` §4 (Export & Archive Creation Specification) |
| **KW-NF-001** | Zero memory leaks and bounded operational memory | `STORAGE_TECHNOLOGY_COMPARISON.md` §3 (Zero-runtime-overhead JSON vs SQLite) |
| **KW-NF-006** | Engine cold startup < 800ms | `STORAGE_TECHNOLOGY_COMPARISON.md` §3 (Direct mmap/serde JSON parsing < 15ms) |
| **KW-NF-012** | Crash isolation and state survivability | `INTEGRITY_SECURITY_AND_DECISIONS.md` §1 (Atomic Replace, `.bak` recovery) |
| **KW-NF-018** | Sandboxed untrusted asset execution | `INTEGRITY_SECURITY_AND_DECISIONS.md` §2 (Sanitization, executable prohibition) |
| **KW-NF-021** | Graceful schema evolution across releases | `SCHEMA_VERSIONING_AND_MIGRATIONS.md` §1-4 (Sequential Migration Pipeline) |
| **ADR-0009** | Storage & Configuration Architecture | Adopted Hybrid JSON + Decentralized Manifest pattern specified across all KW-DAT documents |
| **CTR-04** | Configuration Storage Contract | `DATA_DOMAINS_AND_OWNERSHIP.md` §2.3, `INTEGRITY_SECURITY_AND_DECISIONS.md` §1 |
| **CTR-05** | Library Storage Contract | `DATA_DOMAINS_AND_OWNERSHIP.md` §2.1, `FILE_STORAGE_AND_ASSET_LIFECYCLE.md` §2 |

