# KinetiWall — IPC Boundary and Trust Matrix
**Document ID:** `KW-PRC-005`  
**Deliverable:** Deliverable E — IPC Boundary and Trust Matrix  
**Phase:** 2 — System Architecture & Technical Decisions  
**Step:** 8 of 90 — Multi-Process Boundaries & Component Responsibilities  
**Status:** COMPLETE (Authoritative IPC & Security Baseline)  
**Authority:** Principal Software Architect & IPC Systems Lead  
**Date:** 2026-11-10  

---

## 1. IPC Architectural Charter & Communication Channels

All inter-process communication in KinetiWall occurs over **Windows Named Pipes (`\\.\pipe\kinetiwall-ipc-{SessionId}`)** using a length-prefixed binary framing protocol with typed JSON-RPC 2.0 payloads.

**Security Core:** Local TCP sockets (`127.0.0.1`) are strictly prohibited to avoid firewall warnings and network port enumeration. The Named Pipe is protected by a Windows Object Security DACL granting access exclusively to the current interactive user's logon SID.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                       KINETIWALL IPC PATHWAYS                               │
├─────────────────────┬─────────────────────┬─────────────────────────────────┤
│ Path 1: UI -> Core  │ Path 2: Core -> Play│ Path 3: Play -> Core            │
├─────────────────────┼─────────────────────┼─────────────────────────────────┤
│ • Full User Command │ • Playback Controls │ • Ready Handshake (HWND)        │
│ • State Inquiries   │ • Pause / Resume    │ • Observed State ACK            │
│ • Property Updates  │ • Property Injection│ • Diagnostic Error Strings      │
│ • High Trust (User) │ • Master to Child   │ • Restricted Untrusted Input    │
└─────────────────────┴─────────────────────┴─────────────────────────────────┘
```

---

## 2. Exhaustive IPC Pathway Specifications

The table below catalogs all 5 authorized IPC pathways, defining their trust levels, authorization gates, quotas, and failure modes:

| Pathway ID | Sender Component | Receiver Component | Primary Purpose | Trust Level | Allowed Message Categories | Authorization Gate | Size & Rate Limits | Timeout & Recovery Policy | Security Risk & Mitigations |
|:---|:---|:---|:---|:---:|:---|:---|:---|:---|:---|
| **`IPC-01`** | **Gallery UI** (`ui`) | **Core Controller** (`core`) | User actions: applying wallpapers, updating settings, adjusting sliders. | **Trusted User Space** | `QueryState`, `ApplyWallpaper`, `UpdateProperty`, `SetGlobalPause`, `ImportArchive`. | Core verifies interactive user SID via Named Pipe client token. | Max Payload: **1 MB**; Rate limit: **50 msgs/sec**. | Timeout: **3.0s**. If broken, UI attempts 3 reconnects before showing alert. | Client spoofing: DACL enforces identical logon SID; secondary user tokens rejected. |
| **`IPC-02`** | **Core Controller** (`core`) | **Gallery UI** (`ui`) | State updates: pushing monitor changes, active profiles, and error alerts. | **Trusted User Space** | `StateSync`, `TopologyDelta`, `ErrorNotification`, `EngineShuttingDown`. | Core signs payload with local session token. | Max Payload: **2 MB**; Rate limit: Event-driven. | Non-blocking broadcast. If UI disconnects, drops session gracefully. | UI crash during transmit: Core handles broken pipe without panicking (`FAIL-02`). |
| **`IPC-03`** | **Core Controller** (`core`) | **Video Player** (`video`) | Lifecycle & playback: play, pause, volume, freeze swapchain, terminate. | **Master Supervisor** | `PlayMedia`, `PausePlayback`, `ResumePlayback`, `SetVolume`, `TerminatePlayer`. | Video player accepts commands only from its parent core daemon pipe. | Max Payload: **64 KB**; Rate limit: **30 msgs/sec**. | Timeout: **1.0s**. If player unresponsive, core logs warning. | Malicious injection: Child pipe created with write-only access from parent. |
| **`IPC-04`** | **Core Controller** (`core`) | **Web Player** (`web`) | Content loading, dynamic property updates, and mouse/keyboard event forward. | **Master Supervisor** | `NavigateUrl`, `UpdateDynamicProperty`, `ForwardInputEvent`, `PauseWeb`, `TerminateWeb`.| Web player accepts commands only from parent core daemon pipe. | Max Payload: **256 KB**; Rate limit: **60 msgs/sec** (Input).| Timeout: **1.0s**. Core debounces rapid input events by 16.6ms. | Runaway scripts: Bridge does not expose native win32 execution APIs to DOM. |
| **`IPC-05`** | **Player Subproc** (`video/web`) | **Core Controller** (`core`) | Readiness handshake, state acknowledgment, diagnostic error reporting. | **Sandboxed Subprocess** | `PlayerReady(HWND)`, `PlaybackStateAck`, `LoopBoundaryNotify`, `DiagnosticError`.| Core validates reported HWND belongs to spawned child PID. | Max Payload: **16 KB**; Rate limit: **10 msgs/sec**. | Timeout: **2.0s**. If handshake fails, core terminates and restarts child. | Faked window handles: Core invokes `GetWindowThreadProcessId` to verify HWND. |

---

## 3. Trust Boundary & Data Isolation Matrix

KinetiWall establishes 9 distinct security boundaries, defining what data crosses each boundary and what operations are strictly prohibited:

| Security Entity | Classification | Permitted Inbound Data | Permitted Outbound Data | Enforced Restrictions & Prohibitions |
|:---|:---:|:---|:---|:---|
| **Core Controller** | **Fully Trusted** | Authorized JSON-RPC commands, Win32 OS events, player status ACKs. | State sync arrays, player control commands, atomic config writes. | Never executes untrusted web scripts or decodes video in-process. |
| **Gallery UI** | **Trusted View** | Public application state snapshots, thumbnail image buffers. | User requests: apply wallpaper, update properties, import archives. | **PROHIBITED:** Cannot manipulate desktop HWNDs directly; cannot bypass core. |
| **Video Player** | **Restricted Sandbox** | Media file paths, play/pause commands, volume levels. | Surface `HWND`, playback state ACKs, frame loop notifications. | **PROHIBITED:** Cannot write to `config.json`; cannot spawn child processes. |
| **Web Player** | **Restricted Sandbox** | Sanitized property values, virtual host URLs, input events. | Surface `HWND`, web console logs, dynamic property change events. | **PROHIBITED:** Cannot access local disk outside mapped folder; CORS blocked. |
| **Web JavaScript** | **Untrusted Content** | Property update events injected via `window.chrome.webview`. | Custom property events emitted via `window.chrome.webview.postMessage`.| **PROHIBITED:** No camera/mic; no file access; no process execution; no popups. |
| **Imported Packages** | **Untrusted Input** | User-selected `.zip` or `.kwall` archive files. | Validated, extracted media assets and sanitized `project.json`. | **PROHIBITED:** Traversal paths (`../`) rejected via Zip Slip protection (`SEC-03`).|
| **Local Media Files**| **Passive Assets** | Video containers (.mp4, .mkv, .webm) on local filesystem. | Demuxed compressed video packets read by `libmpv`. | **PROHIBITED:** Executables renamed to .mp4 are rejected by demuxer. |
| **Windows Shell** | **OS Environment** | Desktop window reparenting requests (`SetParent`), `0x052C` message. | `WorkerW` / `Progman` HWNDs, `TaskbarCreated` messages. | **PROHIBITED:** Zero DLL injection; zero memory patching of `explorer.exe`. |
| **Core Audio** | **OS Subsystem** | WASAPI loopback PCM audio frames (read into volatile memory). | 64-band frequency arrays streamed over volatile shared memory. | **PROHIBITED:** Zero disk serialization; zero audio streaming over networks. |

---

## 4. Prohibited IPC Operations & Rejection Enforcement

The following operations are classified as **Security Violations**. Any attempt by a client or player to execute them results in immediate connection termination and diagnostic alert logging:

1. **Arbitrary Process Spawning (`SEC-REJECT-01`):** No IPC message may instruct the core controller to spawn an arbitrary `.exe`, `.bat`, or `.ps1` script.
2. **Arbitrary Filesystem Access (`SEC-REJECT-02`):** No IPC message from a player may request reading or writing files outside `%LOCALAPPDATA%\KinetiWall\wallpapers\{assigned_id}\`.
3. **Elevated Privilege Requests (`SEC-REJECT-03`):** Any command attempting to trigger UAC elevation or run under administrative context is rejected.
4. **Network Egress Commands (`SEC-REJECT-04`):** Any command attempting to open remote TCP/UDP sockets or transmit telemetry is blocked.

