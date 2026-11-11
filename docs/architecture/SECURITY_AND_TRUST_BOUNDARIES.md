# KinetiWall — Security and Trust-Boundary Architecture
**Document ID:** `KW-ARC-006`  
**Deliverable:** Deliverable F — Security and Trust-Boundary Architecture  
**Phase:** 2 — System Architecture & Technical Decisions  
**Step:** 7 of 90 — System Architecture & Architecture Decision Records  
**Status:** COMPLETE (Authoritative Security Architecture)  
**Authority:** Security Architecture & Privacy Engineering Lead  
**Date:** 2026-11-10  

---

## 1. Security Architecture Principles

KinetiWall executes untrusted third-party multimedia, HTML5 scripts, and WebGL code on the Windows desktop. The architecture enforces four non-negotiable security principles:

1. **Least-Privilege Execution (Non-Elevated):** KinetiWall executes strictly within Windows User Mode under the standard Medium Integrity Level. It **never requests UAC elevation** (`requireAdministrator`) and never executes under `LocalSystem`.
2. **Strict Origin Sandboxing:** Web wallpapers execute under a private virtual origin (`https://kinetiwall.local/`) with CORS denial and blocked disk traversal.
3. **Defense-in-Depth IPC Authorization:** Named Pipe connections are restricted by Windows kernel DACLs to the interactive logon SID. Incoming messages are validated against strict Serde JSON schemas.
4. **Transient Audio Privacy:** System loopback audio exists exclusively in volatile memory buffers for real-time FFT frequency calculation. Zero bytes are written to disk or transmitted over networks (`KW-NG-005`).

---

## 2. Security Zoning & Trust Boundaries

The system is partitioned into three distinct security zones separated by kernel-enforced trust boundaries:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    KINETIWALL SECURITY TRUST ZONING                         │
├─────────────────────┬─────────────────────┬─────────────────────────────────┤
│ ZONE 1: UNTRUSTED   │ ZONE 2: SANDBOXED   │ ZONE 3: TRUSTED USER            │
├─────────────────────┼─────────────────────┼─────────────────────────────────┤
│ • Web JavaScript    │ • Web Player Subproc│ • Core Daemon (kinetiwall-core) │
│ • GLSL Shader Code  │ • Video Player Proc │ • Ephemeral UI (kinetiwall-ui)  │
│ • Imported Packages │ • Windows Job Object│ • Local AppData Filesystem      │
│ • Local Media Files │ • Virtual Host Mapp.│ • Named Pipe IPC Server         │
└─────────────────────┴─────────────────────┴─────────────────────────────────┘
```

---

## 3. Trust Boundary Controls & Enforcement Mechanisms

### 3.1 Boundary 1: Untrusted Web Scripts $\longleftrightarrow$ Web Player Container
- **Threat:** Malicious JavaScript attempts to read local files, exfiltrate data, query location, or spawn processes.
- **Enforcement Mechanisms:**
  1. **Virtual Host Directory Mapping:** Mapped via `SetVirtualHostNameToFolderMapping` to `https://kinetiwall.local/` with access kind `COREWEBVIEW2_HOST_RESOURCE_ACCESS_KIND_DENY_CORS`. All cross-origin HTTP/WebSocket requests are rejected.
  2. **Disabled `file://` Scheme:** Loading `file:///` URLs is disabled globally in the WebView2 environment.
  3. **Permission Auto-Denial:** Event handler `PermissionRequested` unconditionally sets `State = COREWEBVIEW2_PERMISSION_STATE_DENY` for Camera, Microphone, Geolocation, and Notification requests.
  4. **Blocked Navigation & Popups:** Event handlers `NewWindowRequested` and `NavigationStarting` cancel all external window opening (`window.open`) and protocol handler activations (`steam://`, `cmd.exe`).
  5. **Controlled Property Bridge:** The only bidirectional communication channel is `window.chrome.webview.postMessage` exchanging structured JSON property events with the player host.

---

### 3.2 Boundary 2: Child Player Processes $\longleftrightarrow$ Core Controller Daemon
- **Threat:** Runaway player processes consuming unlimited RAM/CPU or lingering as background zombies after controller termination.
- **Enforcement Mechanisms:**
  1. **Windows Job Objects:** Every player process is assigned to a Job Object configured with `JOB_OBJECT_LIMIT_KILL_ON_JOB_CLOSE`. When `kinetiwall-core.exe` terminates, the Windows kernel kills all player processes instantaneously.
  2. **Memory Ceiling Quotas:** Web player processes are subjected to a hard 500 MB commit limit via `JobObjectExtendedLimitInformation`. A memory bomb script triggers immediate process termination rather than freezing Windows.
  3. **Bounded Crash Auto-Restart:** If a player crashes 3 times in 60 seconds, the core daemon halts restarts, quarantines the profile, and reverts the monitor to static wallpaper (`KW-F-007`).

---

### 3.3 Boundary 3: Local Named Pipe IPC $\longleftrightarrow$ External Processes
- **Threat:** Secondary user accounts or unprivileged malware connecting to the IPC pipe to manipulate wallpapers or inject commands.
- **Enforcement Mechanisms:**
  1. **Windows NT Discretionary Access Control List (DACL):** The Named Pipe `\\.\pipe\kinetiwall-ipc-{SessionId}` is created with a custom Security Descriptor granting `GENERIC_READ | GENERIC_WRITE` strictly to:
     - The current interactive user logon SID (`TokenUser`).
     - `WinLocalSystemSid` (Windows OS).
     - Network SID explicitly denied.
  2. **Foreign Token Rejection:** Connection attempts from secondary user accounts, low-integrity sandboxes, or remote network endpoints receive `ERROR_ACCESS_DENIED` (5) from the Windows kernel.
  3. **Strict Payload Framing & Schema Validation:** Messages are limited to a 1 MB payload quota. Deserialization is enforced via typed Rust Serde structs. Malformed, truncated, or unparsed packets are discarded immediately.

---

### 3.4 Boundary 4: Imported Wallpaper Archives $\longleftrightarrow$ Local Filesystem
- **Threat:** Malicious ZIP packages exploiting Zip Slip path traversal (`../../Windows/System32/malicious.dll`) to overwrite system files upon import.
- **Enforcement Mechanisms:**
  1. **Path Canonicalization Check:** Before writing any file extracted from a `.kwall` / `.zip` archive, the extraction routine resolves the target path using `std::fs::canonicalize`.
  2. **Prefix Verification:** Asserts that the canonical path begins with `%LOCALAPPDATA%\KinetiWall\wallpapers\{id}\`. If traversal characters (`../` or `..\\`) attempt an escape, the extraction is aborted immediately and the file is deleted.

---

### 3.5 Boundary 5: WASAPI Audio Capture $\longleftrightarrow$ Storage / Network
- **Threat:** Audio loopback capture function recording private user conversations or serializing sound to disk (`KW-NG-005`).
- **Enforcement Mechanisms:**
  1. **Volatile Ring Buffer Allocation:** PCM audio samples captured from WASAPI loopback are placed in a fixed-size circular heap buffer.
  2. **Immediate Mathematical Transformation:** A pure-Rust SIMD worker computes 1024-point FFT magnitudes; raw PCM samples are immediately overwritten on the next audio frame.
  3. **Zero Disk I/O Invariant:** The audio crate contains zero file write APIs (`std::fs::File`, `Write`), zero database bindings, and zero network sockets. Disk write verification is auditable via Process Monitor.

---

## 4. Security Command Authorization Matrix

Every IPC message received by `kinetiwall-core.exe` is validated against the following role-based authorization rules:

| Command Category | Allowed Sender | Required Authorization | Payload Validation Rule |
|:---|:---:|:---:|:---|
| **`QueryStateRequest`** | Gallery UI (`ui`) | Interactive User SID | Empty payload; returns public state. |
| **`ApplyWallpaper`** | Gallery UI (`ui`) | Interactive User SID | Verified `WallpaperId` exists in local library; path within `%LOCALAPPDATA%`. |
| **`UpdateProperty`** | Gallery UI (`ui`) | Interactive User SID | Key matches `project.json` schema; value within defined type and range. |
| **`SetGlobalPause`** | Gallery UI, Tray | Interactive User SID | Boolean flag only. |
| **`PlayerReady`** | Player Subprocess | Child Process Token | Valid `HWND` belonging to spawned PID; matches assigned display. |
| **`PlaybackError`** | Player Subprocess | Child Process Token | Diagnostic error string bounded to 1 KB; sanitized for plain text. |
| **`ArbitraryProcessSpawn`**| **ANY** | **STRICTLY PROHIBITED** | **REJECTED & LOGGED (Security Violation)**. |
| **`ArbitraryFileRead`** | **ANY** | **STRICTLY PROHIBITED** | **REJECTED & LOGGED (Security Violation)**. |

