# COMPREHENSIVE DEPENDENCY INVENTORY
## Document ID: KW-DEP-001 | Phase 2: System Architecture & Technical Decisions

---

### 1. Inventory Summary & Subsystem Allocation

This inventory catalogs 38 evaluated third-party dependencies and runtime components across KinetiWall's architecture. Because the project is currently in the architectural design phase (Steps 1–15), **all dependencies are classified as either APPROVED, PROPOSED, ALTERNATIVE, OPTIONAL, or REJECTED**; none are currently marked as EXISTING in the workspace.

```
+-------------------------------------------------------------------------------------------------------------------------+
|                                        DEPENDENCY SUBSYSTEM OVERVIEW (38 ENTRIES)                                       |
+-----+--------------------------------------+----------+----------+----------+----------+--------------------------------+
| SEC | SUBSYSTEM / DOMAIN                   | APPROVED | PROPOSED | OPTIONAL | REJECTED | PRIMARY ECOSYSTEM              |
+-----+--------------------------------------+----------+----------+----------+----------+--------------------------------+
| 01  | Native Windows & OS API Bindings     | 2        | 0        | 0        | 0        | Cargo (crates.io / Microsoft)  |
| 02  | Async Runtime & Event Loop           | 2        | 1        | 0        | 0        | Cargo (crates.io)              |
| 03  | IPC, Framing & Serialization         | 3        | 0        | 0        | 0        | Cargo (crates.io)              |
| 04  | Video Playback & Media Pipeline      | 1        | 1        | 1        | 1        | Prebuilt Native C DLL / Cargo  |
| 05  | Web Wallpaper Runtime & Sandbox      | 1        | 1        | 0        | 0        | Windows System / Cargo         |
| 06  | UI Frontend Host & Tauri Framework   | 2        | 0        | 0        | 0        | Cargo / npm                    |
| 07  | UI Frontend View (React/Vite/CSS)    | 0        | 6        | 0        | 0        | npm / pnpm                     |
| 08  | Persistence & Storage Strategy       | 1        | 0        | 0        | 1        | Cargo (Pure Rust Serde)        |
| 09  | Audio Capture & Spectrum DSP         | 0        | 2        | 1        | 0        | Cargo (crates.io)              |
| 10  | Image, Thumbnail & Archive Utilities | 3        | 0        | 0        | 0        | Cargo (crates.io)              |
| 11  | Build, Governance, SBOM & Security   | 4        | 3        | 0        | 0        | Cargo Tools / GitHub Actions   |
+-----+--------------------------------------+----------+----------+----------+----------+--------------------------------+
```

---

### 2. Detailed Dependency Catalog

#### 2.1 Subsystem 01: Native Windows & OS API Bindings

##### DEP-01: `windows` (Microsoft Windows API for Rust)
- **Package Identifier**: `crates.io:windows`
- **Current Version**: None (unimplemented) | **Candidate Version**: `0.58`
- **Ecosystem**: Cargo (crates.io) | **Direct/Transitive**: Direct
- **Classification**: **APPROVED (`ADR-0001`, `ADR-0002`)**
- **Purpose**: Provides type-safe native Win32/COM bindings for window creation, message pumps, desktop WorkerW tree navigation, display topology enumeration, process tokens, and DACL security descriptors.
- **Consuming Component**: `kinetiwall-core.exe`, `kinetiwall-player-video.exe`, `kinetiwall-player-web.exe`.
- **Target OS & Arch**: Windows 10/11 (`x86_64-pc-windows-msvc`, `aarch64-pc-windows-msvc`).
- **Maintenance Health**: Actively maintained by Microsoft; updated continuously to track Windows SDK releases.
- **Security History**: Clean; safe abstractions around Win32 pointer types.
- **License**: `MIT OR Apache-2.0` (Permissive).
- **Redistribution Implications**: Include MIT/Apache-2.0 notice in third-party notices file; zero source disclosure requirements.
- **Build Requirements**: MSVC C++ toolchain (`link.exe`).
- **Offline Installation**: Fully compiled into native Rust executable binaries; zero external runtime requirements.
- **Alternative Candidates**: `winapi` (Deprecated, unmaintained since 2021; rejected), `windows-sys` (Low-level raw FFI; adopted as complementary DEP-02).
- **Supporting Sources**: https://github.com/microsoft/windows-rs | https://crates.io/crates/windows

##### DEP-02: `windows-sys` (Raw Windows API Bindings)
- **Package Identifier**: `crates.io:windows-sys`
- **Current Version**: None | **Candidate Version**: `0.58`
- **Ecosystem**: Cargo | **Direct/Transitive**: Direct / Transitive
- **Classification**: **APPROVED**
- **Purpose**: Low-overhead raw Win32 constants and type definitions where full COM abstractions are unnecessary.
- **Consuming Component**: `kinetiwall-core.exe` (DACL definitions, named pipe flags).
- **License**: `MIT OR Apache-2.0` | **Maintenance**: Maintained by Microsoft.

---

#### 2.2 Subsystem 02: Async Runtime & System Integration

##### DEP-03: `tokio` (Asynchronous Runtime)
- **Package Identifier**: `crates.io:tokio`
- **Current Version**: None | **Candidate Version**: `1.40` (Features: `rt-multi-thread`, `net`, `sync`, `time`, `macros`)
- **Ecosystem**: Cargo | **Direct/Transitive**: Direct
- **Classification**: **APPROVED (`ADR-0003`)**
- **Purpose**: Multi-threaded async runtime powering asynchronous Named Pipe IPC servers (`tokio::net::windows::named_pipe`), worker channels, debounce timers, and graceful shutdown orchestration.
- **Consuming Component**: `kinetiwall-core.exe`.
- **Target OS & Arch**: Windows 10/11 (`x86_64`, `aarch64`).
- **Maintenance Health**: Industry standard; backed by Tokio Foundation; thousands of contributors.
- **License**: `MIT` (Permissive).
- **Redistribution Implications**: Attribution notice required. Zero runtime DLLs.
- **Alternative Candidates**: `async-std` (Less mature Windows Named Pipe support; rejected), bare OS threads (`std::thread` only; rejected due to high thread count and synchronization complexity).
- **Supporting Sources**: https://tokio.rs | https://crates.io/crates/tokio

##### DEP-04: `tray-icon` (System Tray Management)
- **Package Identifier**: `crates.io:tray-icon`
- **Current Version**: None | **Candidate Version**: `0.19`
- **Ecosystem**: Cargo | **Direct/Transitive**: Direct
- **Classification**: **APPROVED (`ADR-0002`)**
- **Purpose**: Creates and manages the Windows taskbar notification area (system tray) icon and context menu without requiring a visible window or active UI process.
- **Consuming Component**: `kinetiwall-core.exe`.
- **License**: `MIT OR Apache-2.0` | **Maintenance**: Actively maintained by Tauri ecosystem team.
- **Alternative Candidates**: Hand-rolled `Shell_NotifyIconW` (viable fallback, but `tray-icon` handles DPI scaling and cross-version quirks cleanly).

##### DEP-05: `winit` (Cross-Platform Window Creation)
- **Package Identifier**: `crates.io:winit`
- **Current Version**: None | **Candidate Version**: `0.30`
- **Ecosystem**: Cargo | **Direct/Transitive**: Direct
- **Classification**: **PROPOSED**
- **Purpose**: Optional lightweight window creation for player subprocesses before reparenting under WorkerW.
- **Consuming Component**: `kinetiwall-player-video.exe`.
- **License**: `Apache-2.0` | **Alternative Candidates**: Raw Win32 `CreateWindowExW` via `windows-rs` (Preferred for V1.0 to eliminate winit's large event-loop overhead).

---

#### 2.3 Subsystem 03: IPC, Framing & Serialization

##### DEP-06: `serde` & `serde_json` (Serialization Framework)
- **Package Identifier**: `crates.io:serde` & `crates.io:serde_json`
- **Current Version**: None | **Candidate Version**: `1.0`
- **Ecosystem**: Cargo | **Direct/Transitive**: Direct
- **Classification**: **APPROVED (`ADR-0003`, `ADR-0009`)**
- **Purpose**: High-speed JSON serialization and deserialization for Named Pipe message framing (`IpcEnvelope`), `config.json` persistence, and `project.json` manifest parsing.
- **Consuming Component**: All native Rust binaries (`core`, `players`, `ui` host).
- **License**: `MIT OR Apache-2.0` | **Maintenance**: Exceptional; Rust standard.
- **Performance**: Zero-copy deserialization where applicable; parses 500 KB config in $< 2\text{ms}$.

##### DEP-07: `uuid` (UUID Generation & Parsing)
- **Package Identifier**: `crates.io:uuid`
- **Current Version**: None | **Candidate Version**: `1.10` (Features: `v4`, `serde`)
- **Ecosystem**: Cargo | **Direct/Transitive**: Direct
- **Classification**: **APPROVED (`DEC-DATA-02`)**
- **Purpose**: Generates and parses cryptographically random RFC 4122 UUIDv4 identifiers for wallpaper packages, message correlation, and player instances.
- **Consuming Component**: `kinetiwall-core.exe`, `kinetiwall-ui.exe`.
- **License**: `MIT OR Apache-2.0`.

---

#### 2.4 Subsystem 04: Video Playback & Media Pipeline

##### DEP-08: `libmpv` (mpv Media Player Shared Library)
- **Package Identifier**: Native C/C++ Shared Library (`mpv-2.dll`)
- **Current Version**: None | **Candidate Version**: `mpv 0.38+` (built against FFmpeg 6.x/7.x)
- **Ecosystem**: Prebuilt Native Binary | **Direct/Transitive**: Direct
- **Classification**: **APPROVED (`ADR-0004`, `RC-003`)**
- **Purpose**: Hardware-accelerated video decoding (D3D11VA, NVDEC, QSV), rendering directly to Win32 DirectX swapchains, seamless video looping, frame stepping, and multi-monitor playback.
- **Consuming Component**: `kinetiwall-player-video.exe`.
- **Target OS & Arch**: Windows 10/11 (`x86_64`, `aarch64`).
- **Maintenance Health**: Extremely active open-source project; gold standard for media player embeddings.
- **License**: **LGPL-2.1-or-later** (when built with `--enable-lgpl`; see critical licensing notes in Document KW-DEP-003).
- **Redistribution Implications**: Must be distributed as a separate dynamic shared library (`mpv-2.dll`). LGPL-2.1 source offer and copyright notice must be provided. KinetiWall does NOT statically link libmpv.
- **Offline Installation**: Shipped inside KinetiWall installer (`bin/mpv-2.dll`). Size: ~35–45 MB.
- **Alternative Candidates**: Windows Media Foundation (IMFMediaEngine: Proprietary, patchy MKV/WebM/VP9/AV1 codec support on Windows 10 N editions; rejected), VLC/libVLC (Larger footprint, slower hardware loop sync; rejected).
- **Supporting Sources**: https://mpv.io | https://github.com/mpv-player/mpv

##### DEP-09: `libmpv2` / `mpv` (Rust FFI Wrapper for libmpv)
- **Package Identifier**: `crates.io:libmpv2`
- **Current Version**: None | **Candidate Version**: `4.1`
- **Ecosystem**: Cargo | **Direct/Transitive**: Direct
- **Classification**: **PROPOSED**
- **Purpose**: Safe Rust wrapper around libmpv C API (`mpv_create`, `mpv_set_option`, `mpv_render_context`).
- **License**: `MIT` | **Alternative Candidates**: Hand-crafted safe FFI wrapper (viable alternative if crate has maintenance lag).

---

#### 2.5 Subsystem 05: Web Wallpaper Runtime & Sandboxing

##### DEP-10: Microsoft Edge WebView2 Evergreen Runtime
- **Package Identifier**: Native Windows System Component (`WebView2Loader.dll`)
- **Current Version**: None | **Candidate Version**: Evergreen (Chromium 120+)
- **Ecosystem**: Windows OS Component / Microsoft Redistributable
- **Classification**: **APPROVED (`ADR-0008`, `DEC-SEC-05`)**
- **Purpose**: Hosts isolated Chromium renderers for interactive HTML5, JavaScript, Canvas, and WebGL live wallpapers with virtual host mapping (`https://kinetiwall.local/`).
- **Consuming Component**: `kinetiwall-player-web.exe`.
- **License**: Microsoft Software License Terms for WebView2 (Proprietary, royalty-free redistribution).
- **Redistribution Implications**: Evergreen runtime is pre-installed on Windows 11 and Windows 10 (2004+). For older offline machines, Microsoft provides an Evergreen Bootstrapper or Fixed Version standalone package.
- **Offline Installation**: Pre-installed on >95% of target systems. Standalone installer is bundled for offline deployments (see KW-DEP-007).
- **Supporting Sources**: https://learn.microsoft.com/en-us/microsoft-edge/webview2/

##### DEP-11: `webview2-com` (Rust Bindings for WebView2)
- **Package Identifier**: `crates.io:webview2-com`
- **Current Version**: None | **Candidate Version**: `0.33`
- **Ecosystem**: Cargo | **Direct/Transitive**: Direct
- **Classification**: **PROPOSED**
- **Purpose**: Safe Rust COM interfaces for configuring WebView2 environment, virtual host mapping, permission denials, and `postMessage` event pipelines.
- **License**: `MIT OR Apache-2.0`.

---

#### 2.6 Subsystem 06: UI Frontend Host & Tauri Framework

##### DEP-12: `tauri` v2 (Native Application Host)
- **Package Identifier**: `crates.io:tauri`
- **Current Version**: None | **Candidate Version**: `2.0`
- **Ecosystem**: Cargo | **Direct/Transitive**: Direct
- **Classification**: **APPROVED (`ADR-0007`)**
- **Purpose**: Hosts the on-demand Gallery and Settings interface; provides cross-platform window management, Webview embedding, and system integration.
- **Consuming Component**: `kinetiwall-ui.exe`.
- **License**: `MIT OR Apache-2.0` | **Maintenance**: Actively backed by Tauri Commons.
- **Memory Footprint**: Consumes zero RAM when gallery window is closed (`kinetiwall-ui.exe` terminates on close, leaving only 15 MB `kinetiwall-core.exe` running).

##### DEP-13: `@tauri-apps/api` (Frontend IPC Bridge)
- **Package Identifier**: `npm:@tauri-apps/api`
- **Current Version**: None | **Candidate Version**: `^2.0`
- **Ecosystem**: npm / pnpm | **Direct/Transitive**: Direct
- **Classification**: **APPROVED**
- **Purpose**: JavaScript/TypeScript API for frontend communication with Tauri native host.
- **License**: `MIT OR Apache-2.0`.

---

#### 2.7 Subsystem 07: UI Frontend View Packages (npm/pnpm)

##### DEP-14: `react` & `react-dom`
- **Package Identifier**: `npm:react`, `npm:react-dom`
- **Current Version**: None | **Candidate Version**: `^18.3`
- **Ecosystem**: npm / pnpm | **Direct/Transitive**: Direct
- **Classification**: **PROPOSED**
- **Purpose**: Declarative UI component tree for gallery browsing, monitor layout drag-and-drop, and dynamic property customization sliders.
- **License**: `MIT` (Meta).

##### DEP-15: `vite` (Frontend Build Tool)
- **Package Identifier**: `npm:vite`
- **Candidate Version**: `^5.4` | **Classification**: **PROPOSED** (Build-time only)
- **Purpose**: Ultra-fast bundler for React and TypeScript assets.
- **License**: `MIT`.

##### DEP-16: `tailwindcss` (Utility-First CSS)
- **Package Identifier**: `npm:tailwindcss`
- **Candidate Version**: `^3.4` | **Classification**: **PROPOSED** (Build-time only)
- **Purpose**: Design system styling (Neon Glass aesthetic, responsive dark mode).
- **License**: `MIT`.

##### DEP-17: `lucide-react` (Icon Library)
- **Package Identifier**: `npm:lucide-react`
- **Candidate Version**: `^0.400` | **Classification**: **PROPOSED**
- **Purpose**: Lightweight SVG icons for playback controls, monitors, audio sliders, and settings.
- **License**: `ISC` (Permissive).

##### DEP-18: `zustand` (Frontend State Management)
- **Package Identifier**: `npm:zustand`
- **Candidate Version**: `^4.5` | **Classification**: **PROPOSED**
- **Purpose**: Minimalist, zero-boilerplate client-side UI state store.
- **License**: `MIT`.

##### DEP-19: `typescript` (Static Typing)
- **Package Identifier**: `npm:typescript`
- **Candidate Version**: `^5.5` | **Classification**: **PROPOSED** (Development only)
- **License**: `Apache-2.0`.

---

#### 2.8 Subsystem 08: Persistence & Storage Strategy

##### DEP-20: Pure Rust Hybrid JSON Engine (`serde_json` + `std::fs`)
- **Package Identifier**: Internal Rust Abstraction (backed by DEP-06)
- **Classification**: **APPROVED (`ADR-0009`, `DEC-DATA-01`)**
- **Purpose**: Manages `config.json` atomic replacement (`MoveFileExW`) and decentralized package `project.json` manifests.
- **License**: Built into application code (MIT/Apache-2.0 compatible).

##### DEP-21: `rusqlite` (Embedded SQLite Integration)
- **Package Identifier**: `crates.io:rusqlite`
- **Classification**: **REJECTED FROM V1.0 (`ADR-0009`, `DEC-DATA-01`)**
- **Rationale for Rejection**: Unnecessary for V1.0 libraries (< 1,000 items); introduces C FFI compilation overhead (~1.5 MB binary bloat), creates Windows multi-process file locking hazards (`SQLITE_BUSY`), and increases corruption blast radius. Evaluated as a potential future secondary read cache for V2.0+ if library scale exceeds 1,500 items.

---

#### 2.9 Subsystem 09: Audio Capture & Spectrum DSP

##### DEP-22: `cpal` (Cross-Platform Audio Library)
- **Package Identifier**: `crates.io:cpal`
- **Candidate Version**: `0.15` | **Classification**: **PROPOSED**
- **Purpose**: Safe WASAPI loopback audio stream capture for audio-reactive wallpapers.
- **License**: `Apache-2.0`.
- **Alternative Candidates**: Direct Win32 WASAPI via `windows-rs` (eliminates `cpal` dependency entirely; evaluated as preferred alternative in Phase 3).

##### DEP-23: `rustfft` (Fast Fourier Transform DSP)
- **Package Identifier**: `crates.io:rustfft`
- **Candidate Version**: `6.2` | **Classification**: **PROPOSED**
- **Purpose**: Pure Rust, SIMD-accelerated Fast Fourier Transform converting captured PCM audio samples into 16 or 32 normalized frequency bands.
- **License**: `MIT OR Apache-2.0` (Permissive).
- **Safety**: 100% pure Rust; zero C dependencies; immune to C buffer overflows.

---

#### 2.10 Subsystem 10: Image, Thumbnail & Archive Utilities

##### DEP-24: `image` (Pure Rust Image Processing)
- **Package Identifier**: `crates.io:image`
- **Candidate Version**: `0.25` (Features: `png`, `webp`, `jpeg`)
- **Classification**: **APPROVED (`ADR-0009`)**
- **Purpose**: Decodes and resizes video keyframe captures into 320x180 and 640x360 thumbnails.
- **License**: `MIT OR Apache-2.0`.

##### DEP-25: `zip` (ZIP Archive Decompression)
- **Package Identifier**: `crates.io:zip`
- **Candidate Version**: `2.2` (Default features without bzip2/zstd)
- **Classification**: **APPROVED (`KW-DAT-005`)**
- **Purpose**: Decompresses imported `.kwp` / `.zip` wallpaper packages with stream size checks and Zip Slip path filters.
- **License**: `MIT`.

##### DEP-26: `sha2` (Cryptographic Hashing)
- **Package Identifier**: `crates.io:sha2`
- **Candidate Version**: `0.10` | **Classification**: **APPROVED (`DEC-DATA-02`)**
- **Purpose**: Computes SHA-256 content hashes of packages and assets for deduplication and tamper detection.
- **License**: `MIT OR Apache-2.0`.

---

#### 2.11 Subsystem 11: Diagnostics, Build, SBOM & Governance Tools

##### DEP-27: `tracing` & `tracing-subscriber` (Structured Logging)
- **Candidate Version**: `0.1` / `0.3` | **Classification**: **APPROVED**
- **Purpose**: Structured, high-performance diagnostics with automatic redaction of sensitive paths.
- **License**: `MIT`.

##### DEP-28: `tracing-appender` (Rolling File Logging)
- **Candidate Version**: `0.2` | **Classification**: **APPROVED**
- **Purpose**: Non-blocking log file rotation with 7-day automated pruning.
- **License**: `MIT`.

##### DEP-29: `cargo-about` (Attribution Notice Generator)
- **Candidate Version**: `0.6` | **Classification**: **APPROVED (CI Tool)**
- **Purpose**: Scans Cargo workspace and generates comprehensive HTML/markdown third-party license notices.
- **License**: `Apache-2.0`.

##### DEP-30: `cargo-cyclonedx` (SBOM Generator)
- **Candidate Version**: `0.5` | **Classification**: **APPROVED (CI Tool)**
- **Purpose**: Emits compliant CycloneDX JSON Software Bill of Materials.
- **License**: `Apache-2.0`.

##### DEP-31: `cargo-audit` (Security Advisory Scanner)
- **Candidate Version**: `0.20` | **Classification**: **APPROVED (CI Tool)**
- **Purpose**: Audits `Cargo.lock` against RustSec Advisory Database.
- **License**: `MIT OR Apache-2.0`.

