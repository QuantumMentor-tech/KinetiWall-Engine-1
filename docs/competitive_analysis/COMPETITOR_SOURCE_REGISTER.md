# KinetiWall — Competitor Source Evidence Register

**Document Identifier:** KW-COMP-007  
**Document Type:** Formal Competitor Evidence Register (Deliverable G)  
**Current Phase:** Phase 1 — Project Research, Requirements & Feasibility  
**Step:** Step 3 of 90 — Competitor Analysis & Product Differentiation  
**Status:** Approved Source Register  
**Date of Research:** 2026-11-08  

---

## 1. Register Overview & Confidence Standard

This register establishes a fully traceable catalog of primary sources, official product documentation, verified repository codebases, and issue trackers used to substantiate the competitor analysis and differentiation strategy in Step 3.

### Evidence Classification Scheme:
- **`[VERIFIED]`**: Directly extracted from official documentation, vendor manuals, or official repository source code.
- **`[OBSERVED]`**: Verified through reproducible execution, binary inspection, or verified maintainer issue reports.
- **`[REPORTED]`**: Corroborated across multiple credible community discussions or technical reviews.
- **`[INFERRED]`**: Derived logically from underlying platform architecture or toolchain constraints.

---

## 2. Bibliographic Source Records

### CSR-001: Wallpaper Engine Official Documentation & Store Specifications
- **Product Name:** Wallpaper Engine (Kristjan Skutta)
- **Specific Capabilities Investigated:** Supported formats (MP4, WebM, AVI), Direct3D 11 rendering pipeline, embedded Chromium/CEF web runtime, pause-on-fullscreen behavior, multi-monitor configuration, Steam Workshop dependency, system requirements.
- **Source Titles & URLs:**
  1. *Wallpaper Engine Official Manual & FAQ:* https://www.wallpaperengine.io/
  2. *Wallpaper Engine Steam Store Page:* https://store.steampowered.com/app/431960/Wallpaper_Engine/
  3. *Wallpaper Engine Web Wallpaper Technical Guide:* https://steamcommunity.com/sharedfiles/filedetails/?id=1082779273
- **Publication / Access Date:** Continuous Vendor Updates / Accessed: 2026-11-08
- **Evidence Type:** `[VERIFIED]` (Official Vendor Documentation & Store Specifications).
- **Confidence Level:** **HIGH**
- **Limitations & Unresolved Questions:** Internal Direct3D swapchain implementation details and proprietary `.pkg` scene format specifications are closed-source.
- **Project Relevance & Impact:** Establishes the commercial benchmark for video looping quality, multi-monitor spanning, and pause rules. Validates the strategic opportunity to offer a non-Steam, local-first alternative.

---

### CSR-002: Lively Wallpaper Official Repository & Windows 11 24H2 Issue Logs
- **Product Name:** Lively Wallpaper (Dani John / `rocksdanister`)
- **Specific Capabilities Investigated:** .NET 8 / WinUI 3 architecture, `mpv` player integration, sub-desktop `WorkerW` parenting via `0x052C`, WASAPI audio visualizer, GPL-3.0 licensing, and Windows 11 24H2 stability issues.
- **Source Titles & URLs:**
  1. *Lively Wallpaper GitHub Repository:* https://github.com/rocksdanister/lively
  2. *Lively Wallpaper Official Website:* https://www.rocksdanister.com/lively/
  3. *Windows 11 24H2 Desktop Composition Issues (#2500+):* https://github.com/rocksdanister/lively/issues
- **Publication / Access Date:** 2024–2026 / Accessed: 2026-11-08
- **Evidence Type:** `[VERIFIED]` & `[OBSERVED]` (Open-Source Codebase Audit & Maintainer Issue Logs).
- **Confidence Level:** **HIGH**
- **Limitations & Unresolved Questions:** WinUI 3 composition surfaces change between Windows App SDK updates, requiring continuous monitoring of upstream Microsoft SDK releases.
- **Project Relevance & Impact:** Substantiates the critical finding that naive `0x052C` WorkerW attachment fails on Windows 11 24H2 when desktop icons are hidden, directly driving KinetiWall's dual-mode attachment requirement (`UDR-002`, `TR-001`). Confirms the 150–250 MB memory bloat caused by .NET 8 / WinUI 3.

---

### CSR-003: Aura COSMIC Desktop Live Wallpaper Engine (Rust Technical Reference)
- **Product Name:** Aura (`antwny/aura`)
- **Specific Capabilities Investigated:** Pure Rust architecture, `tokio` asynchronous event loops, `mpv` / `ffmpeg` embedding, fallback static poster states, and Linux Wayland layer-shell integration.
- **Source Titles & URLs:**
  1. *Aura GitHub Repository:* https://github.com/antwny/aura
  2. *Aura Official Documentation:* https://antwny.github.io/aura/
- **Publication / Access Date:** 2024–2026 / Accessed: 2026-11-08
- **Evidence Type:** `[VERIFIED]` (Open-Source Rust Repository Inspection).
- **Confidence Level:** **HIGH**
- **Limitations & Unresolved Questions:** Targets Linux / Wayland compositors (Sway, Hyprland, COSMIC); zero compatibility with the Windows Win32 shell or DWM.
- **Project Relevance & Impact:** Confirms that Aura is **not a Windows competitor**, resolving prior research ambiguity. Serves as a valuable architectural reference for Rust systems patterns, non-blocking supervisor loops, and fallback poster mechanics.

---

### CSR-004: AutoWall Minimalist Media Player Wrapper
- **Product Name:** AutoWall (SegoCode)
- **Specific Capabilities Investigated:** Lightweight CLI wrapper around `mpv.exe` and `weebp.exe`, sub-desktop window parenting, minimal process memory footprint, and lack of management UI.
- **Source Titles & URLs:**
  1. *AutoWall GitHub Repository:* https://github.com/SegoCode/AutoWall
- **Publication / Access Date:** 2023–2025 / Accessed: 2026-11-08
- **Evidence Type:** `[VERIFIED]` (Repository Code Inspection).
- **Confidence Level:** **HIGH**
- **Limitations & Unresolved Questions:** Lacks multi-monitor topology handling, pause rules, audio visualization, settings persistence, or library management.
- **Project Relevance & Impact:** Proves that an mpv-based live wallpaper wrapper can achieve <20 MB memory, but highlights that without multi-monitor management, fullscreen pause detection, and a gallery interface, user adoption is severely constrained.

---

### CSR-005: Giantapp LiveWallpaper (巨应壁纸)
- **Product Name:** Giantapp LiveWallpaper (`GiantappMan/livewallpaper`)
- **Specific Capabilities Investigated:** Dual licensing (GPL-2.0 / LGPL-2.1), TypeScript / C# hybrid codebase, built-in Chinese CDN community integration, and Microsoft Store packaging.
- **Source Titles & URLs:**
  1. *Giantapp LiveWallpaper GitHub Repository:* https://github.com/GiantappMan/livewallpaper
  2. *Giantapp Official Portal:* https://livewallpaper.giantapp.cn/en
- **Publication / Access Date:** 2024–2026 / Accessed: 2026-11-08
- **Evidence Type:** `[VERIFIED]` & `[REPORTED]` (Repository Code & Community Documentation).
- **Confidence Level:** **MEDIUM**
- **Limitations & Unresolved Questions:** Core community features rely heavily on regional Chinese infrastructure; international performance is limited.
- **Project Relevance & Impact:** Highlights the architectural hazards of combining disparate runtimes (TypeScript, C#, PowerShell), reinforcing KinetiWall's commitment to a unified native Rust core architecture.

