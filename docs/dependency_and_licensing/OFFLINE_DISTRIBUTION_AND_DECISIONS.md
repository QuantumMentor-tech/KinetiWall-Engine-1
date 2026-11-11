# OFFLINE DISTRIBUTION ASSESSMENT & DEPENDENCY DECISION REGISTER
## Document ID: KW-DEP-007 | Phase 2: System Architecture & Technical Decisions

---

### 1. Offline Distribution Assessment (Deliverable H)

KinetiWall is architected as an **offline-first, locally sovereign desktop application**. Ordinary live wallpaper playback, monitor assignment, property customization, and library management MUST NEVER require an active Internet connection, user account login, or cloud telemetry service.

#### 1.1 Installer Packaging Profiles

To accommodate both standard consumer environments and air-gapped enterprise workstations, KinetiWall defines two distinct release distribution profiles:

```
+---------------------------------------------------------------------------------------------------------+
|                                    KINETIWALL INSTALLATION PROFILES                                     |
+--------------------------+------------------------------------+-----------------------------------------+
| PROFILE ATTRIBUTE        | STANDARD INSTALLER (DEFAULT)       | FULL OFFLINE / ENTERPRISE INSTALLER     |
+--------------------------+------------------------------------+-----------------------------------------+
| Target Environment       | Standard Windows 10/11 Desktops    | Air-gapped machines, Enterprise, Offline|
| Target Distribution Size | **~65 – 85 MB**                    | **~220 – 260 MB**                       |
| Bundled Components       | - kinetiwall-core.exe              | - kinetiwall-core.exe                   |
|                          | - kinetiwall-ui.exe (Tauri)        | - kinetiwall-ui.exe (Tauri)             |
|                          | - player subprocesses              | - player subprocesses                   |
|                          | - mpv-2.dll (LGPL build, ~40 MB)   | - mpv-2.dll (LGPL build, ~40 MB)        |
|                          | - WebView2Loader.dll               | - WebView2Loader.dll                    |
|                          | - Bundled default wallpapers       | - Bundled default wallpapers            |
|                          |                                    | - WebView2 Standalone Installer (160 MB)|
| WebView2 Strategy        | Utilizes OS Evergreen Runtime;     | Executes bundled offline standalone     |
|                          | invokes bootstrapper if missing    | installer if Evergreen runtime missing  |
| Network Requirement      | Zero during and after install*     | **Absolute Zero Network Requirement**   |
+--------------------------+------------------------------------+-----------------------------------------+
```

*\*Note*: The Standard Installer requires network access only if the host Windows system is an older Windows 10 build that somehow lacks the Microsoft Edge WebView2 Evergreen runtime (< 5% of machines).

#### 1.2 Disk Footprint & Runtime Resource Budget
- **Installed Disk Footprint**: ~160 MB (binaries, DLLs, assets, and base wallpapers).
- **Idle Memory Footprint**: Core Controller ~15 MB RAM; Video Player ~35–50 MB RAM per active display; Web Player ~60–80 MB RAM per active display.
- **Zero Runtime Prerequisite Downloads**: Everything necessary to decode MP4/MKV/WebM videos and execute standard HTML5 wallpapers is installed locally on the disk.

---

### 2. Open Dependency Decision Register (`DEC-DEP-01` through `DEC-DEP-08`)

```
+--------------------------------------------------------------------------------------------------------------------------+
|                                           DEPENDENCY DECISION REGISTER (STEP 11)                                         |
+-------------+-------------------------------------------------------+----------+-----------------------------------------+
| DECISION ID | TITLE / SUMMARY                                       | STATUS   | ARCHITECTURAL RATIONALE                 |
+-------------+-------------------------------------------------------+----------+-----------------------------------------+
| DEC-DEP-01  | Microsoft `windows-rs` as Exclusive Win32 Abstraction | APPROVED | Official Microsoft crate, type safety   |
| DEC-DEP-02  | Multi-Threaded `tokio` Async Runtime for Core         | APPROVED | High-performance async Named Pipes & IPC|
| DEC-DEP-03  | Mandatory LGPL-2.1 Build Configuration for `libmpv`   | APPROVED | Prevents GPL contamination of codebase  |
| DEC-DEP-04  | Dynamic Linking & Process Isolation for `mpv-2.dll`   | APPROVED | Satisfies LGPL relinking & FSF isolation|
| DEC-DEP-05  | Exclusion of SQLite RDBMS from V1.0 Baseline          | APPROVED | Eliminates C FFI bloat & file locks     |
| DEC-DEP-06  | Prohibition of Third-Party Native Executable Plugins  | APPROVED | Prevents remote code execution exploits |
| DEC-DEP-07  | OWASP CycloneDX JSON as Authoritative SBOM Standard   | APPROVED | Industry-standard AppSec & tool support |
| DEC-DEP-08  | Mandatory Version-Controlled Lockfiles (Cargo & pnpm) | APPROVED | Prevents supply-chain drift & attacks   |
+-------------+-------------------------------------------------------+----------+-----------------------------------------+
```

#### Detailed Decision Records:

- **DEC-DEP-01: Microsoft `windows-rs` as Exclusive Win32 Abstraction**
  - *Context*: Selecting Win32 bindings crate (`winapi` vs `windows-rs` vs raw extern C).
  - *Decision*: Adopt official `windows` (0.58+) and `windows-sys` crates maintained by Microsoft.
  - *Consequences*: Full coverage of modern Windows 11 APIs, COM interfaces, type-safe error handling, zero legacy `winapi` maintenance risk.

- **DEC-DEP-02: Multi-Threaded `tokio` Async Runtime for Core Controller**
  - *Context*: Coordinating asynchronous Named Pipe IPC servers and timer-driven health checks.
  - *Decision*: Adopt `tokio` (1.40+) with explicitly scoped feature flags (`rt-multi-thread`, `net`, `sync`, `time`).
  - *Consequences*: Sub-millisecond IPC responsiveness, robust timer handling, zero blocking of Windows desktop messages.

- **DEC-DEP-03: Mandatory LGPL-2.1 Build Configuration for `libmpv`**
  - *Context*: Ensuring video decoding engine does not impose viral copyleft obligations on KinetiWall.
  - *Decision*: Mandate that bundled `mpv-2.dll` is compiled strictly with `--enable-lgpl` and `-Dgpl=false`.
  - *Consequences*: The distributed binary is legally licensed under LGPL-2.1-or-later; commercial and permissive redistribution is fully compliant.

- **DEC-DEP-04: Dynamic Linking & Process Isolation for `mpv-2.dll`**
  - *Context*: Satisfying LGPL Section 6 relinking requirements and isolating media decoder faults.
  - *Decision*: Distribute `mpv-2.dll` as an independent dynamic library loaded strictly by `kinetiwall-player-video.exe`.
  - *Consequences*: Users can replace the DLL; media crashes cannot bring down the Core controller; complete architectural isolation.

- **DEC-DEP-05: Exclusion of SQLite RDBMS from V1.0 Baseline**
  - *Context*: Evaluating embedded database (`rusqlite`) vs Hybrid JSON architecture.
  - *Decision*: Reject SQLite for V1.0 (`ADR-0009`, `DEC-DATA-01`). Adopt transactional `config.json` + folder manifests.
  - *Consequences*: Eliminates 1.5 MB binary bloat, avoids C FFI compiler requirements, eliminates Windows file-locking conflicts across processes.

- **DEC-DEP-06: Prohibition of Third-Party Native Executable Plugins**
  - *Context*: Addressing requests for arbitrary `.exe` live wallpaper integration.
  - *Decision*: Exclude native `.exe` wallpaper execution from V1.0 (`KW-NG-002`). All wallpapers must be video or web.
  - *Consequences*: Blocks malware execution vectors permanently; maintains strict sandboxing.

- **DEC-DEP-07: OWASP CycloneDX JSON as Authoritative SBOM Standard**
  - *Context*: Selecting Software Bill of Materials standard for enterprise releases.
  - *Decision*: Standardize on CycloneDX JSON generated via `cargo-cyclonedx`.
  - *Consequences*: Seamless integration with automated vulnerability scanners (Dependency-Track, OSV).

- **DEC-DEP-08: Mandatory Version-Controlled Lockfiles (`Cargo.lock` & `pnpm-lock.yaml`)**
  - *Context*: Guarding against dependency drift and supply-chain poisoning.
  - *Decision*: Commit lockfiles to Git; enforce `cargo build --locked` and `pnpm install --frozen-lockfile` in all CI pipelines.
  - *Consequences*: 100% deterministic, reproducible build inputs.

---

### 3. Dependency Risk Register

```
+--------------------------------------------------------------------------------------------------------------------------+
|                                             DEPENDENCY RISK REGISTER SUMMARY                                             |
+-----------+----------------------------------------------------+----------+--------+-------------------------------------+
| RISK ID   | TITLE / DESCRIPTION                                | SEVERITY | STATUS | PLANNED MITIGATION                  |
+-----------+----------------------------------------------------+----------+--------+-------------------------------------+
| KW-RD-001 | Unintentional GPL contamination via libmpv build   | Critical | Open   | Mandate --enable-lgpl build flag in |
|           | flags in third-party distributions.                |          |        | CI; automate license string check.  |
| KW-RD-002 | WebView2 Evergreen runtime missing on legacy Win10 | High     | Open   | Provide Full Offline installer with |
|           | offline enterprise workstations.                   |          |        | bundled standalone runtime setup.   |
| KW-RD-003 | RUSTSEC advisory disclosed in transitive crate     | Medium   | Open   | Daily automated cargo-audit; apply  |
|           | blocking release CI build.                         |          |        | [patch.crates-io] emergency hotfix. |
| KW-RD-004 | MediaFoundation vs libmpv codec compatibility gap  | Medium   | Closed | Adopted libmpv across all platforms |
|           | on Windows 10 N editions (missing Media Feature).  |          |        | eliminating OS codec dependency.    |
| KW-RD-005 | Supply-chain poisoning via compromised npm package | High     | Open   | Enforce pnpm --frozen-lockfile,     |
|           | in frontend React dependency tree.                 |          |        | scoped packages, and pnpm audit.    |
+-----------+----------------------------------------------------+----------+--------+-------------------------------------+
```

