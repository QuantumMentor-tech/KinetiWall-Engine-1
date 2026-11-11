# KinetiWall — Runtime & Dependency Compatibility Matrix

**Document Identifier:** KW-POL-004  
**Document Type:** Systems-Level Dependency & Runtime Evaluation (Deliverable D)  
**Current Phase:** Phase 1 — Project Research, Requirements & Feasibility  
**Step:** Step 4 of 90 — Windows Compatibility & Platform Support Policy  
**Status:** Approved Runtime Matrix Baseline  
**Date of Research:** 2026-11-08  

---

## 1. Runtime Architecture & Evaluation Overview

KinetiWall requires several third-party runtime subsystems and native operating system components to support its multi-process live wallpaper architecture.

This document evaluates the compatibility, deployment models, distribution footprints, offline operability, licensing constraints, and failure modes across:
1. **Microsoft Edge WebView2 Runtime** (HTML5, WebGL, and interactive web wallpapers).
2. **`libmpv` & FFmpeg Shared Media Libraries** (Hardware-accelerated video playback).
3. **Windows Audio Session API (WASAPI) & DSP Runtimes** (Audio-reactive visualizations).
4. **Direct3D 11 & DXGI Swapchain Subsystems** (Video presentation and surface composition).
5. **Rust Target Toolchains & C/C++ CRT Runtimes** (Compiler, linker, and MSVC runtime dependencies).

---

## 2. Comprehensive Runtime & Dependency Matrix

| Component / Subsystem | Provider / Source | Supported Windows Versions | Architecture Support | Deployment Model | Offline Footprint | Licensing Obligation | Failure Mode & Fallback |
| :--- | :--- | :--- | :--- | :--- | :---: | :--- | :--- |
| **Microsoft Edge WebView2 (Evergreen)** | Microsoft Corporation | Windows 10 (19041+) & Windows 11 | x86-64 & ARM64 (Native) | System-preinstalled on modern Windows; auto-updated by Edge | **0 MB** (Shared system runtime) | Proprietary Microsoft (Free redistribution via Evergreen bootstrapper) | Registry detection failure: UI surfaces download prompt; core continues (`UDR-006`). |
| **Microsoft Edge WebView2 (Fixed Version)** | Microsoft Corporation | Windows 10 & Windows 11 | x86-64 & ARM64 | Bundled standalone directory in specialized offline installers | **~180 MB** (Compressed package) | Proprietary Microsoft (Redistributable under WebView2 license) | Runtime self-contained; zero external network dependencies. |
| **`libmpv-2.dll` (Shared Dynamic Library)** | mpv.io community / FFmpeg | Windows 10 & Windows 11 | x86-64 (Primary); ARM64 (Requires custom build) | Bundled dynamically linked shared `.dll` in application directory | **~22 MB** (DLLs + dependencies) | **GNU LGPL v2.1+** (Strict `--enable-lgpl --disable-gpl` build) | DLL load failure: Logs error, falls back to static desktop image (`KW-F-015`). |
| **Direct3D 11 Runtime (`d3d11.dll`)** | Microsoft / Windows OS | Windows 7 SP1 through Win 11 24H2 | x86-64 & ARM64 | Native Windows Operating System component | **0 MB** (Built-in) | Proprietary Microsoft (OS component) | Feature level < 10_0: Software WARP rasterizer fallback (`d3d10warp.dll`). |
| **DXGI Swapchains (`dxgi.dll`)** | Microsoft / Windows OS | Windows 8 through Win 11 24H2 | x86-64 & ARM64 | Native Windows Operating System component | **0 MB** (Built-in) | Proprietary Microsoft (OS component) | `DXGI_ERROR_DEVICE_RESET`: Re-create swapchains on wake within 2.0s (`KW-NF-008`). |
| **Windows Core Audio / WASAPI** | Microsoft / Windows OS | Windows Vista through Win 11 24H2 | x86-64 & ARM64 | Native Windows Operating System component | **0 MB** (Built-in) | Proprietary Microsoft (OS component) | `AUDCLNT_E_DEVICE_INVALIDATED`: Re-bind to default endpoint within 500ms (`KW-F-033`). |
| **`rustfft` DSP Library** | Open-source Rust crate | All platforms | Pure Rust (No C dependencies) | Statically compiled inside `kinetiwall-core` binary | **<500 KB** (Binary code size) | MIT / Apache-2.0 (Permissive) | Silence detected: Thread sleeps until audio signal returns. |
| **Microsoft Visual C++ CRT (`VCRUNTIME140.dll`)** | Microsoft Corporation | Windows 10 & Windows 11 | x86-64 & ARM64 | Statically linked (`crt-static`) or bundled dynamic DLLs | **~2 MB** | Microsoft Visual Studio EULA (Redistributable) | Missing CRT: Statically link MSVC CRT in release builds to eliminate dependency. |

---

## 3. Microsoft Edge WebView2 Deep-Dive & Offline Distribution

### 3.1 Evergreen vs. Fixed Version Strategy
- **The Core Dilemma:** KinetiWall requires WebView2 for interactive HTML5/WebGL wallpapers (`KW-F-016`, `KW-F-017`). The project is also committed to **100% offline operability** (`KW-F-045`) and maintaining a compact installer (<35 MB bare, `KW-NF-006`).
- **Empirical Availability Analysis:**
  - On Windows 11, the Evergreen WebView2 Runtime is preinstalled on 100% of consumer and enterprise devices out-of-the-box (`PSR-006`).
  - On Windows 10, Microsoft deployed the Evergreen WebView2 Runtime via standard Windows Updates to all supported versions (2004, 20H2, 21H1, 21H2, 22H2) starting in 2022 (`PSR-006`).
  - Only air-gapped enterprise machines or legacy Windows 10 1903/1909 installations lack Evergreen WebView2 by default.
- **Strategic Dual-Installer Decision (`UDR-006`):**
  1. **Standard Desktop Installer (`~56 MB` with sample media):** Targets >98% of users. Checks the Windows Registry for the system-installed Evergreen runtime. Leverages the preinstalled browser engine with 0 MB runtime overhead.
  2. **Offline Enterprise Bundle (`~236 MB`):** Targets air-gapped workstations. Bundles the Microsoft WebView2 Fixed Version package directly inside the installer, guaranteeing 100% offline installation without an internet connection.

### 3.2 Runtime Detection & Safe Failure Protocol
When `kinetiwall-player-web.exe` initializes:
```rust
fn detect_webview2_runtime() -> bool {
    // 1. Check 64-bit Registry Key
    const REG_KEY: &str = r"SOFTWARE\WOW6432Node\Microsoft\EdgeUpdate\Clients\{F3017226-FE2A-4295-8BDF-00C3A9A7E4C5}";
    if let Ok(key) = RegKey::open(HKEY_LOCAL_MACHINE, REG_KEY, KEY_READ) {
        if let Ok(version) = key.get_string("pv") {
            if !version.is_empty() && version != "0.0.0.0" {
                return true; // Valid Evergreen Runtime found
            }
        }
    }
    // 2. Check local bundled Fixed Version directory
    Path::new("runtimes/webview2_fixed").exists()
}
```
If WebView2 is missing, KinetiWall:
- Disables web wallpaper selection in the UI with an informative warning badge.
- Leaves video wallpaper playback and desktop integration 100% operational.
- Provides a direct, non-blocking notification: *"Interactive web wallpapers require the Microsoft Edge WebView2 runtime. Click here for offline installer guidance."*

---

## 4. Media Pipeline & `libmpv` Dependency Verification

### 4.1 Compiler Toolchain & C-FFI Compatibility
- **Toolchain Standard:** `libmpv-2.dll` must be compiled using MSVC (`cl.exe`) or modern MinGW-w64 with MSVC C-ABI compatibility to interface seamlessly with Rust's `x86_64-pc-windows-msvc` target.
- **Dynamic Linking Contract:** `kinetiwall-player-video.exe` uses dynamic C-FFI bindings (via `libmpv2` / Win32 `LoadLibraryExW`). It dynamically imports entry points:
  - `mpv_create`
  - `mpv_initialize`
  - `mpv_set_option_string`
  - `mpv_command`
  - `mpv_render_context_create`
  - `mpv_render_context_render`
- **Static Linking Prohibition:** To avoid copyleft GPL contamination (`UDR-005`, `KW-RES-001`), static linking (`.lib` / `.a`) against `libmpv` or `FFmpeg` is strictly forbidden.

### 4.2 Hardware Decoding Codec Compatibility Matrix

| Video Codec | Container Format | Direct3D 11 Decoder Context | Windows 10 (19041+) Support | Windows 11 Support | Fallback if HW Decode Fails |
| :--- | :---: | :---: | :---: | :---: | :--- |
| **H.264 / AVC** | `.mp4`, `.avi` | `d3d11va` (DXVA2_ModeH264_VLD) | **Universal Hardware Support** | **Universal Hardware Support** | Seamless CPU decode (`dav1d` / `libavcodec`) |
| **H.265 / HEVC** | `.mp4`, `.mkv` | `d3d11va` (DXVA2_ModeHEVC_VLD) | Supported on modern GPUs (GTX 960+, RX 400+) | Supported on modern GPUs | Seamless CPU decode |
| **VP8** | `.webm` | Software / Hybrid GPU | Supported | Supported | CPU decode (`libvpx`) |
| **VP9** | `.webm`, `.mkv` | `d3d11va` (DXVA2_ModeVP9_VLD) | Supported on modern GPUs (GTX 1060+, RX 500+) | Universal on modern hardware | Seamless CPU decode |
| **AV1** | `.mp4`, `.webm` | `d3d11va` (DXVA2_ModeAV1_VLD) | Supported on RTX 30-series+, RX 6000+, Intel Arc | Universal on modern hardware | High-speed CPU decode via `dav1d` |

---

## 5. Direct3D 11 Feature Levels & Graphics Hardware Requirements

To guarantee smooth swapchain presentation across modern and legacy GPUs without graphics driver panics:
1. **Minimum Feature Level:** `D3D_FEATURE_LEVEL_10_0` (Supports Direct3D 11 API on older Intel HD Graphics 4000 and NVIDIA GeForce 400-series).
2. **Recommended Feature Level:** `D3D_FEATURE_LEVEL_11_1` (Supports high-efficiency Direct3D 11.1 swapchains, DXGI flip-model presentation `DXGI_SWAP_EFFECT_FLIP_DISCARD`, and low-latency texture sharing).
3. **Multi-GPU / Hybrid Laptop Switching:** On dual-GPU laptops (Intel/AMD integrated GPU + NVIDIA discrete GPU), `kinetiwall-core` selects the **Integrated GPU (iGPU)** by default for wallpaper decoding to minimize power consumption, leaving the discrete GPU completely idle for gaming (`KW-NF-003`, `KW-NF-005`).

