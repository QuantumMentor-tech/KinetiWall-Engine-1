# KinetiWall — Technical Source Evidence Register

**Document Identifier:** KW-RES-002  
**Document Type:** Formal Source Evidence Register (Deliverable B)  
**Current Phase:** Phase 1 — Project Research, Requirements & Feasibility  
**Step:** Step 2 of 90 — Technical Research & Source Verification  
**Status:** Approved Source Register  
**Date of Research:** 2026-11-08  

---

## 1. Register Overview & Evidence Taxonomy

This source register provides a comprehensive, traceable inventory of primary technical sources, official API specifications, maintainer issue threads, and independent engineering benchmarks used to validate KinetiWall's architecture during Step 2.

### Confidence Level Definitions:
- **HIGH:** Substantiated by official vendor documentation (Microsoft Learn, Rust-lang, Khronos) or confirmed maintainer implementation code.
- **MEDIUM:** Substantiated by credible engineering investigations, open-source project issues, or reproducible community benchmarks with minor limitations.
- **LOW:** Incomplete, indirect, or disputed evidence requiring further empirical validation.
- **UNVERIFIED:** Claim not successfully substantiated; treated as an unconfirmed assumption.

---

## 2. Primary Technical Sources & Evidence Catalog

### SR-001: Windows Desktop Window Composition (`Progman` & `WorkerW`)
- **Claim Investigated:** Sending undocumented message `0x052C` to `Progman` spawns a `WorkerW` window beneath desktop icons (`SHELLDLL_DefView`).
- **Source Title:** Windows Desktop Compositor & Shell Architecture / Windows 11 24H2 Desktop Hierarchy
- **Direct Source URL:** https://learn.microsoft.com/en-us/windows/win32/shell/shell-window & https://github.com/rocksdanister/lively/issues
- **Relevant Section/Reference:** Win32 Window Hierarchy, Message `0x052C`, `SHELLDLL_DefView` parenting
- **Publication / Access Date:** Update: 2024 / Accessed: 2026-11-08
- **What Evidence Establishes:** In Windows 10 and 11, `SendMessageTimeoutW(hProgman, 0x052C, 0, 0, ...)` forces Explorer to create a `WorkerW` window. However, in Windows 11 24H2, changes to desktop composition cause `WorkerW` to detach or vanish if desktop icons are toggled off.
- **Limitations & Conflicting Evidence:** Undocumented internal behavior. Microsoft provides no public API contract or guarantee of backwards compatibility across future Windows updates.
- **Confidence Level:** **HIGH** (Extensively proven across multiple open-source and commercial engines, with confirmed 24H2 caveats).
- **Consequence for KinetiWall:** Must implement robust enumeration, handle invalidation checks, and fallback mechanisms (`UDR-002`, `KW-F-001`, `KW-F-003`).

---

### SR-002: Windows Explorer Restart Recovery via `TaskbarCreated`
- **Claim Investigated:** Desktop shell restarts can be detected reliably using the `TaskbarCreated` registered window message.
- **Source Title:** Microsoft Learn: `RegisterWindowMessageW` & Taskbar Notifications
- **Direct Source URL:** https://learn.microsoft.com/en-us/windows/win32/shell/taskbar
- **Relevant Section/Reference:** "Taskbar Creation Notification", `RegisterWindowMessage(TEXT("TaskbarCreated"))`
- **Publication / Access Date:** Official Documentation / Accessed: 2026-11-08
- **What Evidence Establishes:** When `explorer.exe` restarts, it broadcasts the system-wide registered message `TaskbarCreated` to all top-level windows. Applications intercepting this message can safely re-register tray icons and re-bind shell hooks.
- **Limitations & Conflicting Evidence:** During Explorer bootup, window hierarchies can take 200–500ms to stabilize; rapid attachment immediately upon receiving the message can race with `Progman` initialization.
- **Confidence Level:** **HIGH** (Officially documented Microsoft API).
- **Consequence for KinetiWall:** Core daemon must register `TaskbarCreated` with a 500ms debounce timer before re-attaching player windows (`KW-F-002`, `KW-F-041`).

---

### SR-003: Rust Windows Bindings (`windows` and `windows-sys` Crates)
- **Claim Investigated:** The official `windows` and `windows-sys` crates provide safe and zero-overhead Win32/COM bindings suitable for multi-process desktop engines.
- **Source Title:** Microsoft GitHub: `microsoft/windows-rs`
- **Direct Source URL:** https://github.com/microsoft/windows-rs
- **Relevant Section/Reference:** `windows` v0.58 / `windows-sys` v0.59 documentation
- **Publication / Access Date:** 2024–2026 / Accessed: 2026-11-08
- **What Evidence Establishes:** `windows-sys` provides raw C-FFI bindings with near-zero compilation overhead for Win32 calls. `windows` provides full COM interface generation, typed `HRESULT` returns, and smart pointer wrappers.
- **Limitations & Conflicting Evidence:** Mixing `windows` and `windows-sys` types requires manual pointer casting; crate dependencies must be segregated per crate module.
- **Confidence Level:** **HIGH** (Official Microsoft-maintained Rust project).
- **Consequence for KinetiWall:** Adopt `windows-sys` for the high-performance core daemon and `windows` for COM-heavy player modules (`KW-NF-025`, `KW-NF-026`).

---

### SR-004: Windows Job Objects & Process Tree Termination
- **Claim Investigated:** Windows Job Objects configured with `JOB_OBJECT_LIMIT_KILL_ON_JOB_CLOSE` automatically terminate all child processes when the parent process exits.
- **Source Title:** Microsoft Learn: `JOBOBJECT_EXTENDED_LIMIT_INFORMATION` structure
- **Direct Source URL:** https://learn.microsoft.com/en-us/windows/win32/api/winnt/ns-winnt-jobobject_extended_limit_information
- **Relevant Section/Reference:** `JOB_OBJECT_LIMIT_KILL_ON_JOB_CLOSE` (0x00002000), `SetInformationJobObject`
- **Publication / Access Date:** Official Documentation / Accessed: 2026-11-08
- **What Evidence Establishes:** The Windows kernel terminates all assigned processes in a job object when the last handle to the job object is closed, completely eliminating zombie processes upon unexpected parent crashes.
- **Limitations & Conflicting Evidence:** If the job object handle is configured with `bInheritHandle = TRUE`, child processes inherit the handle, preventing the kernel handle count from reaching zero on parent exit. Handle inheritance must be disabled.
- **Confidence Level:** **HIGH** (Core Windows NT kernel capability).
- **Consequence for KinetiWall:** All player subprocesses must be assigned to an unnamed Job Object with non-inheritable handles (`KW-NF-010`, `UDR-003`).

---

### SR-005: Tokio Windows Named Pipes with Custom Security Descriptors (DACL)
- **Claim Investigated:** Tokio Named Pipe servers can enforce strict single-user SID access controls using raw security attributes.
- **Source Title:** Tokio Documentation & Microsoft Learn: Named Pipe Security and Access Rights
- **Direct Source URL:** https://docs.rs/tokio/latest/tokio/net/windows/named_pipe/ & https://learn.microsoft.com/en-us/windows/win32/ipc/named-pipe-security-and-access-rights
- **Relevant Section/Reference:** `ServerOptions::create_with_security_attributes_raw`, `ConvertStringSecurityDescriptorToSecurityDescriptorW`
- **Publication / Access Date:** 2024–2026 / Accessed: 2026-11-08
- **What Evidence Establishes:** Tokio's `create_with_security_attributes_raw` allows passing a custom `SECURITY_ATTRIBUTES` struct with an SDDL string granting access exclusively to the current user SID and Local System, blocking unauthorized cross-user IPC access.
- **Limitations & Conflicting Evidence:** Security descriptor buffers must remain pinned in memory for the duration of pipe instantiation.
- **Confidence Level:** **HIGH** (Documented in Tokio and Win32 Security API).
- **Consequence for KinetiWall:** IPC listener must initialize with user SID-restricted DACL and reject remote clients (`KW-F-008`, `KW-NF-012`).

---

### SR-006: Hardware Video Decoding and Seamless Looping in `libmpv`
- **Claim Investigated:** `libmpv` with `hwdec=d3d11va`, `vo=gpu-next`, and `loop-file=inf` achieves hardware-accelerated, seamless video looping on Windows.
- **Source Title:** mpv.io Reference Manual: Video Output Drivers & Hardware Decoding
- **Direct Source URL:** https://mpv.io/manual/master/
- **Relevant Section/Reference:** `--hwdec=d3d11va`, `--vo=gpu-next`, `--loop-file=inf`
- **Publication / Access Date:** Continuous / Accessed: 2026-11-08
- **What Evidence Establishes:** Direct3D 11 video acceleration offloads decoding to dedicated GPU hardware. Using `vo=gpu-next` eliminates 1-frame black flashes across loop boundaries by maintaining continuous swapchain texture presentation.
- **Limitations & Conflicting Evidence:** `vo=gpu` exhibits occasional black frame glitches during D3D11 device flush on loop boundaries; `vo=gpu-next` resolves this but requires modern graphics driver support.
- **Confidence Level:** **HIGH** (Documented in mpv manual and confirmed by community benchmarks).
- **Consequence for KinetiWall:** Video player engine must standardize on `libmpv` with `vo=gpu-next` and `hwdec=d3d11va` (`KW-F-012`, `KW-F-013`, `UDR-004`).

---

### SR-007: Microsoft Edge WebView2 Virtual Host Mapping & Security
- **Claim Investigated:** WebView2's `SetVirtualHostNameToFolderMapping` isolates web wallpapers from local filesystem access while enabling relative asset loading.
- **Source Title:** Microsoft Learn: `ICoreWebView2_3::SetVirtualHostNameToFolderMapping` Method
- **Direct Source URL:** https://learn.microsoft.com/en-us/microsoft-edge/webview2/reference/win32/icorewebview2_3#setvirtualhostnametofoldermapping
- **Relevant Section/Reference:** `COREWEBVIEW2_HOST_RESOURCE_ACCESS_KIND_DENY_CORS`
- **Publication / Access Date:** Official Documentation / Accessed: 2026-11-08
- **What Evidence Establishes:** Mapping a local folder to a virtual origin (`https://kinetiwall.local/`) provides an isolated HTTPS origin. Scripts cannot read files outside the folder or make cross-origin filesystem requests.
- **Limitations & Conflicting Evidence:** If the access kind is set to `ALLOW`, external web pages embedded via iframe could potentially read local assets; `DENY_CORS` or `DENY` must be enforced.
- **Confidence Level:** **HIGH** (Official Microsoft documentation).
- **Consequence for KinetiWall:** Web player will use virtual host mapping with `DENY_CORS` and disable Camera/Microphone/Geolocation permissions (`KW-F-018`, `KW-NF-016`, `UDR-010`).

---

### SR-008: Event-Driven Window Hooks & Fullscreen Detection
- **Claim Investigated:** WinEvent hooks (`EVENT_SYSTEM_FOREGROUND`) combined with `SHQueryUserNotificationState` provide low-overhead fullscreen and occlusion tracking.
- **Source Title:** Microsoft Learn: `SetWinEventHook` & `SHQueryUserNotificationState`
- **Direct Source URL:** https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-setwineventhook & https://learn.microsoft.com/en-us/windows/win32/api/shellapi/nf-shellapi-shqueryusernotificationstate
- **Relevant Section/Reference:** `EVENT_SYSTEM_FOREGROUND`, `QUNS_RUNNING_D3D_FULL_SCREEN`
- **Publication / Access Date:** Official Documentation / Accessed: 2026-11-08
- **What Evidence Establishes:** Event hooks eliminate polling loops. `QUNS_RUNNING_D3D_FULL_SCREEN` detects exclusive fullscreen games accurately.
- **Limitations & Conflicting Evidence:** Borderless fullscreen games may report `QUNS_BUSY` instead of `QUNS_RUNNING_D3D_FULL_SCREEN`; window boundary comparison (`GetWindowRect` matching monitor bounds) is required as a secondary check.
- **Confidence Level:** **HIGH** (Documented Win32 APIs).
- **Consequence for KinetiWall:** Implement dual-tier detection: `SHQueryUserNotificationState` for exclusive 3D, and geometry matching for borderless games (`KW-F-025`, `KW-F-028`, `UDR-007`).

---

### SR-009: WASAPI Audio Loopback Capture in Rust
- **Claim Investigated:** Windows Audio Session API in loopback mode (`AUDCLNT_STREAMFLAGS_LOOPBACK`) enables system audio capture without virtual audio cables.
- **Source Title:** Microsoft Learn: Loopback Recording & CPAL Documentation
- **Direct Source URL:** https://learn.microsoft.com/en-us/windows/win32/coreaudio/loopback-recording & https://docs.rs/cpal/
- **Relevant Section/Reference:** `AUDCLNT_STREAMFLAGS_LOOPBACK`, CPAL input stream on output device
- **Publication / Access Date:** Official Documentation / Accessed: 2026-11-08
- **What Evidence Establishes:** WASAPI loopback streams PCM float audio directly from the default multimedia render endpoint.
- **Limitations & Conflicting Evidence:** When default audio output switches (e.g. plugging in Bluetooth headphones), the active stream fails with `AUDCLNT_E_DEVICE_INVALIDATED`. Dynamic re-initialization via `IMMNotificationClient` is required.
- **Confidence Level:** **HIGH** (Documented Windows CoreAudio API).
- **Consequence for KinetiWall:** Implement WASAPI loopback capture with dynamic device invalidation listeners and volatile memory processing (`KW-F-031`, `KW-F-033`, `KW-NG-005`).

---

### SR-010: Zip Slip Vulnerability in Archive Extraction
- **Claim Investigated:** Extracting untrusted wallpaper packages without path validation exposes the system to directory traversal attacks ("Zip Slip").
- **Source Title:** Snyk Security Research: Zip Slip Vulnerability
- **Direct Source URL:** https://snyk.io/research/zip-slip-vulnerability/
- **Relevant Section/Reference:** Archive directory traversal via `../` sequences
- **Publication / Access Date:** Snyk Vulnerability Report / Accessed: 2026-11-08
- **What Evidence Establishes:** Malicious archives containing relative directory traversal sequences (`../../evil.exe`) can overwrite files outside the extraction directory if extracted with naive path concatenation.
- **Limitations & Conflicting Evidence:** None; well-documented standard security vulnerability.
- **Confidence Level:** **HIGH** (Industry standard security finding).
- **Consequence for KinetiWall:** Wallpaper package extractor must canonicalize paths and verify destination directory boundaries before writing (`KW-NF-014`).

