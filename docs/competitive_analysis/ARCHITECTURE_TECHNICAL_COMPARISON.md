# KinetiWall — Architecture & Technical Comparison

**Document Identifier:** KW-COMP-003  
**Document Type:** In-Depth Systems Architecture & Engineering Comparison (Deliverable C)  
**Current Phase:** Phase 1 — Project Research, Requirements & Feasibility  
**Step:** Step 3 of 90 — Competitor Analysis & Product Differentiation  
**Status:** Approved Technical Architecture Comparison  
**Date of Research:** 2026-11-08  

---

## 1. Architectural Scope & Purpose

This document provides a rigorous, code-level and systems-level comparative analysis of the engineering approaches used by **Wallpaper Engine**, **Lively Wallpaper**, and **KinetiWall**. The objective is to identify why specific technical decisions were made, evaluate their performance and stability trade-offs, and ground KinetiWall's architectural choices in empirical evidence.

---

## 2. Deep-Dive Comparative Subsystem Analysis

### 2.1 UI Framework & Process Lifecycle Architecture

```
+---------------------------------------------------------------------------------------------------+
|                               UI PROCESS LIFECYCLE COMPARISON                                     |
+---------------------------------------------------------------------------------------------------+
| LIVELY WALLPAPER (C# / .NET 8 / WinUI 3)                                                          |
| [ Tray Icon ] <---> [ WinUI 3 Window ] (Kept resident in memory or minimized)                    |
| RAM Overhead: ~150–250 MB constantly allocated (.NET CLR Heap + WinUI 3 Composition Surfaces)     |
+---------------------------------------------------------------------------------------------------+
| WALLPAPER ENGINE (Native C++ / Direct3D 11)                                                       |
| [ Tray Icon ] <---> [ Native Dialog / CEF Window ] (Minimized to tray, background threads active) |
| RAM Overhead: ~30–60 MB resident in memory                                                        |
+---------------------------------------------------------------------------------------------------+
| KINETIWALL (Native Rust Core Daemon + Ephemeral Tauri v2 UI)                                      |
| [ kinetiwall-core.exe ] (Pure Rust Daemon: ~8–12 MB RAM, resident in Notification Area)           |
|         |                                                                                         |
|         | User clicks "Open Gallery" -> Spawns kinetiwall-ui.exe (~70 MB RAM)                     |
|         | User closes Gallery Window -> kinetiwall-ui.exe EXECUTES std::process::exit(0)          |
|         v                                                                                         |
| [ kinetiwall-core.exe ] (Memory immediately returns to ~8–12 MB; UI memory is 100% freed)        |
+---------------------------------------------------------------------------------------------------+
```

- **Lively Wallpaper:** Built on .NET 8 and WinUI 3 (Windows App SDK). While WinUI 3 offers modern Fluent visual styling, the .NET Garbage Collector and Windows App SDK composition layers retain substantial heaps. Even when minimized to the notification area, Lively's background memory footprint rarely drops below 150 MB.
- **Wallpaper Engine:** Implements a custom native C++ application interface with embedded Chromium/CEF for web dialogs. Highly optimized, maintaining approximately 30–60 MB idle RAM.
- **KinetiWall Architecture:** Adopts a **Strict Ephemeral UI Pattern** (`KW-F-007`, `KW-NG-001`). The core supervising daemon (`kinetiwall-core.exe`) is written in pure Rust using `windows-sys` with no UI framework dependencies, consuming **under 10 MB RAM**. The gallery interface (`kinetiwall-ui.exe`) is launched as an on-demand child process. When closed by the user, the UI process terminates completely, freeing all memory.

---

### 2.2 Desktop Window Attachment & Windows 11 24H2 Resilience

| Product | Window Discovery Method | Message Injection | Window Parenting Target | Windows 11 24H2 Resilience |
| :--- | :--- | :--- | :--- | :--- |
| **Wallpaper Engine** | `FindWindowW(L"Progman", NULL)` | `0x052C` to `Progman` | Attaches to `WorkerW` or parents directly into `Progman` | **High** (Custom internal logic with fallback paths) |
| **Lively Wallpaper** | `FindWindowExW` hierarchy search | `SendMessageTimeoutW(0x052C)` | Attaches child window to sibling `WorkerW` | **Vulnerable** (Regression issues on 24H2 when desktop icons hidden) |
| **KinetiWall** | `FindWindowW` + `EnumWindows` | `0x052C` with 500ms debounce | **Dual-Mode:** WorkerW primary; direct `Progman` fallback beneath `SHELLDLL_DefView` | **Engineered for 24H2** (Explicit detection of hidden icon state) |

- **Technical Nuance:** In Windows 11 24H2, if the user right-clicks the desktop and unchecks *View -> Show desktop icons*, Windows Explorer destroys or alters `SHELLDLL_DefView`. In Lively Wallpaper, this causes the enumeration routine to fail to locate the expected `WorkerW` sibling, triggering infinite refresh loops or a solid grey background (`CSR-002`).
- **KinetiWall Solution:** KinetiWall's attachment routine explicitly inspects `SHELLDLL_DefView` presence. If icons are hidden, it does not rely on the `0x052C` sibling `WorkerW`, but parents directly into `Progman` positioned beneath desktop layers using `SetWindowPos(hPlayerWnd, HWND_BOTTOM, ...)` with a 500ms debounce guard.

---

### 2.3 Video Playback Engine & Gapless Looping Integration

- **Wallpaper Engine:** Uses a proprietary Direct3D 11 rendering engine with custom media source readers and DXVA2 hardware acceleration. It employs a ping-pong decoder buffer to achieve virtually seamless video looping without black flashes.
- **Lively Wallpaper:** Evaluated Windows Media Foundation (WMF) and LibVLC before standardizing on an embedded `mpv` player (`libmpv`). WMF was abandoned because WMF flushes Direct3D video processors upon reaching the end of a media stream, producing a visible 1-frame black pause or stutter.
- **KinetiWall Architecture:** Standardizes on embedded `libmpv-2.dll` via dynamic C-FFI bindings (`KW-F-010`, `KW-F-012`). By configuring `vo=gpu-next`, `hwdec=d3d11va`, and `loop-file=inf`, KinetiWall bypasses user-space seek latencies, maintaining active Direct3D swapchain presentation across loop boundaries with inter-frame jitter well below 16.6 ms (`KW-F-013`).

---

### 2.4 Web Wallpaper Sandboxing & Security Boundaries

| Security Parameter | Wallpaper Engine | Lively Wallpaper | KinetiWall (Planned Baseline) |
| :--- | :--- | :--- | :--- |
| **Web Runtime** | Embedded Chromium (CEF) | CEF / WebView2 | Microsoft Edge WebView2 COM |
| **Protocol Scheme** | `http://` / custom scheme | `http://localhost:port` or `file:///` | Virtual Host `https://kinetiwall.local/` |
| **Local File Access** | Restricted by CEF flags | Accessible if `file:///` used | **Strictly Denied** (`COREWEBVIEW2_HOST_RESOURCE_ACCESS_KIND_DENY_CORS`) |
| **Camera & Microphone** | Blocked | Blocked | **Explicitly Denied via API Handler** |
| **Process Permissions** | Standard User | Standard User | **Standard User (`asInvoker`), No Elevation** |

- **Security Hazard in Competitors:** Earlier implementations of open-source live wallpaper engines served local web content via raw `file:///` URIs or unauthenticated local HTTP servers (`127.0.0.1:port`). This allowed malicious JavaScript inside an imported web wallpaper to probe the local filesystem or exploit local loopback ports.
- **KinetiWall Architecture:** Implements modern Microsoft Edge WebView2 virtual host mapping (`ICoreWebView2_3::SetVirtualHostNameToFolderMapping`). Web wallpapers load strictly from `https://kinetiwall.local/index.html`. All parent directory traversal (`../../`), local file execution, camera, microphone, and geolocation permissions are blocked at the engine boundary (`KW-F-018`, `KW-NF-016`).

---

### 2.5 Process Supervision & Crash Isolation

- **Lively Wallpaper:** Spawns helper processes for playback. If a player crashes, the main C# daemon detects the process exit event and can attempt a restart, but child processes are not bound by Windows Job Objects, risking orphaned processes if the main app crashes.
- **Wallpaper Engine:** Uses robust internal watchdog monitoring across its 32-bit and 64-bit helper binaries.
- **KinetiWall Architecture:** Enforces kernel-level supervision via **Windows Job Objects** (`KW-NF-010`). All player processes are assigned to an unnamed Job Object configured with `JOB_OBJECT_LIMIT_KILL_ON_JOB_CLOSE` and non-inheritable handles (`bInheritHandle = FALSE`). If `kinetiwall-core.exe` terminates or is forcefully killed in Task Manager, the Windows NT kernel automatically terminates all child player processes within 100 ms, completely eliminating zombie processes.

---

### 2.6 Inter-Process Communication (IPC) & Access Control

- **Lively Wallpaper:** Communicates across processes using Windows Named Pipes and Win32 window messages (`WM_COPYDATA`).
- **Wallpaper Engine:** Uses shared memory mapped files and named pipes for low-latency frame and state passing.
- **KinetiWall Architecture:** Implements asynchronous Windows Named Pipes using Tokio (`tokio::net::windows::named_pipe`). Crucially, KinetiWall configures a custom Discretionary Access Control List (DACL) restricting connection rights exclusively to the current user's Security Identifier (SID) and Local System (`KW-NF-012`). This blocks unauthorized cross-user command injection or pipe hijacking on shared or multi-user Windows workstations.

---

### 2.7 Audio Capture & Real-Time Visualization Subsystem

| Parameter | Wallpaper Engine | Lively Wallpaper | KinetiWall (Planned Baseline) |
| :--- | :--- | :--- | :--- |
| **Audio Capture API** | WASAPI Loopback Capture | WASAPI Loopback Capture | WASAPI Loopback (`AUDCLNT_STREAMFLAGS_LOOPBACK`) |
| **FFT Processing** | In-engine native DSP | C# / NAudio DSP pipeline | Rust in-memory FFT (`rustfft`, Hann window) |
| **Dynamic Device Change**| Handles default audio switch | Re-initializes on device change | `IMMNotificationClient::OnDefaultDeviceChanged` |
| **Silence / Idle Sleep** | Auto-sleeps during silence | Sleeps during silence | Enters idle wait state after >3s silence |
| **Audio Privacy Guarantee**| Local only, no recording | Local only, no recording | **Strict volatile memory processing; zero disk writes (`KW-NG-005`)** |

- **Technical Parity & Privacy:** All three engines correctly leverage WASAPI loopback capture rather than requiring virtual audio cables. KinetiWall formalizes audio privacy as an explicit non-goal (`KW-NG-005`), guaranteeing that PCM audio buffers reside strictly in transient volatile memory and are never serialized to disk or transmitted over the network.

