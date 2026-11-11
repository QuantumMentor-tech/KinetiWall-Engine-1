# SECURITY THREAT MODEL & STRIDE ANALYSIS
## Document ID: KW-SEC-004 | Phase 2: System Architecture & Technical Decisions

---

### 1. Threat Modeling Methodology & Protected Assets

KinetiWall implements the industry-standard **STRIDE** methodology (Spoofing, Tampering, Repudiation, Information Disclosure, Denial of Service, Elevation of Privilege) combined with a quantitative Risk Scoring Matrix ($R = L \times I$) established in **Step 6 (KW-R-001..020)**.

#### 1.1 Protected Assets Inventory

| Asset ID | Asset Name | Description | Confidentiality | Integrity | Availability | Owner |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **AST-01** | **Core Control Authority** | Execution rights for engine lifecycle, process supervision, and system operations. | High | Critical | Critical | `kinetiwall-core.exe` |
| **AST-02** | **Desktop Window State** | Windows shell window hierarchy (`WorkerW`, `Progman`, desktop icons). | Low | High | Critical | Windows Shell / Core |
| **AST-03** | **Application Settings** | User preferences, autostart keys, and monitor assignments in `config.json`. | Low | High | High | `ConfigurationRepo` |
| **AST-04** | **Host Filesystem** | User documents, system files, and AppData directories outside wallpapers. | Critical | Critical | High | Windows OS |
| **AST-05** | **Audio Stream Privacy** | Audio playing on the system captured for visualization. | Critical | High | Medium | Windows Core Audio |
| **AST-06** | **Process Handles** | Win32 native handles to child processes (`HANDLE` with `PROCESS_TERMINATE`). | Medium | Critical | High | `LifecycleSupervisor` |
| **AST-07** | **GPU / Display Stability** | Direct3D11/12 swapchains and display driver operational stability. | Low | High | Critical | DirectX / GPU Driver |

---

### 2. Attack Surfaces and Threat Actors

#### 2.1 Threat Actors
1. **Hostile Untrusted Content Author**: An external creator publishing a malicious `.kwp` / `.zip` live wallpaper designed to compromise systems upon import.
2. **Untrusted Web Script (Zone 5)**: Malicious JavaScript embedded in an interactive wallpaper attempting sandbox breakout, data exfiltration, or crypto-mining.
3. **Malicious Local Process**: An unauthorized third-party process running in the same user session attempting to hijack KinetiWall's IPC pipe to manipulate files or processes.
4. **Malfunctioning / Crash-Prone Subprocess**: A legitimate player subprocess experiencing an unhandled exception, infinite loop, or resource leak.

---

### 3. Formal Threat Register (`KW-T-001` through `KW-T-020`)

```
+-------------------------------------------------------------------------------------------------------------------------+
|                                              KINETIWALL THREAT REGISTER SUMMARY                                         |
+----------+----------------------------------------------------+----------+------------+--------+------------------------+
| THREAT ID| TITLE / SUMMARY                                    | STRIDE   | LIKELIHOOD | IMPACT | RISK MITIGATION STATUS |
+----------+----------------------------------------------------+----------+------------+--------+------------------------+
| KW-T-001 | Named Pipe Client Impersonation Token Theft        | ElevPriv | Medium     | High   | MITIGATED (Design)     |
| KW-T-002 | Unauthorized Local Process Connecting to Pipe      | Spoofing | Medium     | High   | MITIGATED (Design)     |
| KW-T-003 | Cross-Session IPC Pipe Hijacking                   | Spoofing | Low        | High   | MITIGATED (Design)     |
| KW-T-004 | Zip Slip Path Traversal during Package Extraction  | Tampering| High       | Crit   | MITIGATED (Design)     |
| KW-T-005 | Decompression Archive Bomb (Disk / CPU Exhaustion) | DoS      | Medium     | High   | MITIGATED (Design)     |
| KW-T-006 | Malicious Executable Dropper via Wallpaper Package | ElevPriv | High       | Crit   | MITIGATED (Design)     |
| KW-T-007 | Web Wallpaper Script Escaping WebView2 Sandbox     | ElevPriv | Low        | Crit   | MITIGATED (Design)     |
| KW-T-008 | Web Wallpaper Accessing Files via file:/// URI     | InfoDisc | High       | High   | MITIGATED (Design)     |
| KW-T-009 | Web Wallpaper Exfiltrating Data via Outbound Fetch | InfoDisc | Medium     | High   | MITIGATED (Design)     |
| KW-T-010 | Raw Audio Eavesdropping via WASAPI Loopback Capture| InfoDisc | Medium     | Crit   | MITIGATED (Design)     |
| KW-T-011 | Malformed IPC Message Flooding / CPU Saturation    | DoS      | Medium     | Medium | MITIGATED (Design)     |
| KW-T-012 | Giant Oversized IPC Payload Memory Exhaustion      | DoS      | Medium     | High   | MITIGATED (Design)     |
| KW-T-013 | Stale Player Instance Command Confusion (PID Reuse)| Tampering| Medium     | Medium | MITIGATED (Design)     |
| KW-T-014 | Desktop Window Hierarchy Corruption                | DoS      | Medium     | High   | MITIGATED (Design)     |
| KW-T-015 | Config File Corrupt Overwrite via Sudden Power Loss| Tampering| High       | High   | MITIGATED (Design)     |
| KW-T-016 | Autostart Registry Shell Injection                 | ElevPriv | Low        | Crit   | MITIGATED (Design)     |
| KW-T-017 | GPU VRAM Exhaustion via Infinite WebGL Textures    | DoS      | High       | Medium | MITIGATED (Design)     |
| KW-T-018 | User PII or Token Exposure in Diagnostic Logs      | InfoDisc | Medium     | Medium | MITIGATED (Design)     |
| KW-T-019 | Broken Player Process Becoming Orphaned Zombie     | DoS      | High       | Medium | MITIGATED (Design)     |
| KW-T-020 | Replay Attack on Non-Idempotent IPC Commands       | Tampering| Low        | Medium | MITIGATED (Design)     |
+----------+----------------------------------------------------+----------+------------+--------+------------------------+
```

---

### 4. Detailed Threat Records

#### KW-T-001: Named Pipe Client Impersonation Token Theft
- **STRIDE**: Elevation of Privilege | **Target**: `AST-01` (Core Controller)
- **Attack Vector**: A malicious client connects to the Core named pipe and induces the server to call `ImpersonateNamedPipeClient`, stealing the caller's administrative token.
- **Preconditions**: Core calls Win32 impersonation APIs.
- **Consequence**: Full privilege escalation to the token level of connected clients.
- **Mitigation**:
  1. Complete code ban: `ImpersonateNamedPipeClient` is prohibited in all KinetiWall source code.
  2. All clients connect with `SECURITY_SQOS_PRESENT | SECURITY_IDENTIFICATION`, cryptographically disallowing impersonation at the NT kernel boundary.
- **Residual Risk**: Low. Status: **MITIGATED IN DESIGN**.

#### KW-T-002: Unauthorized Local Process Connecting to Pipe
- **STRIDE**: Spoofing / Tampering | **Target**: `AST-01` (Core Authority)
- **Attack Vector**: A random unprivileged background process running under the user session connects to `\\.\pipe\kinetiwall-core-*` and issues commands.
- **Mitigation**:
  1. Win32 SDDL DACL restricts pipe access strictly to the owner user SID.
  2. Server queries `GetNamedPipeClientProcessId` upon connection and verifies the client PID against the trusted `ProcessRegistry`. Unregistered PIDs are disconnected immediately.
- **Residual Risk**: Low. Status: **MITIGATED IN DESIGN**.

#### KW-T-004: Zip Slip Path Traversal during Package Extraction
- **STRIDE**: Tampering / Elevation of Privilege | **Target**: `AST-04` (Host Filesystem)
- **Attack Vector**: A wallpaper `.zip` contains filenames like `../../../../AppData/Roaming/Startup/malicious.bat`.
- **Consequence**: Arbitrary file overwrite outside the application sandbox.
- **Mitigation**: Path canonicalization sandbox check. Any archive entry with `..`, absolute drive prefixes (`C:`), or resolving outside `%LOCALAPPDATA%\KinetiWall\cache\tmp\import_{uuid}\` triggers immediate extraction abort and sandbox purge.
- **Residual Risk**: Negligible. Status: **MITIGATED IN DESIGN**.

#### KW-T-006: Malicious Executable Dropper via Wallpaper Package
- **STRIDE**: Elevation of Privilege | **Target**: `AST-01`, `AST-04`
- **Attack Vector**: Wallpaper package bundles a `.exe`, `.dll`, or `.ps1` script to execute arbitrary native code.
- **Mitigation**: Strict extension blacklist during ingest. Any package containing `.exe`, `.dll`, `.bat`, `.cmd`, `.ps1`, `.vbs`, `.sys`, `.msi`, `.scr` is rejected at validation gate with `ERR_SECURITY_VIOLATION`.
- **Residual Risk**: Low. Status: **MITIGATED IN DESIGN**.

#### KW-T-007: Web Wallpaper Script Escaping WebView2 Sandbox
- **STRIDE**: Elevation of Privilege | **Target**: `AST-01`, `AST-04`
- **Attack Vector**: Vulnerability in Chromium/Edge rendering engine allows remote code execution or native API invocation.
- **Mitigation**:
  1. WebView2 runs with full sandboxing enabled.
  2. Zero native COM or C++ objects injected into JavaScript runtime.
  3. `CoreWebView2Settings.AreHostObjectsAllowed = false`.
  4. Only message channel is string-based `window.chrome.webview.postMessage`.
  5. Renderer process runs at Low Mandatory Integrity level.
- **Residual Risk**: Bound by Chromium zero-day surface (Low). Status: **MITIGATED IN DESIGN**.

#### KW-T-008: Web Wallpaper Accessing Files via `file:///` URI
- **STRIDE**: Information Disclosure | **Target**: `AST-04` (Host Filesystem)
- **Attack Vector**: Web wallpaper JavaScript issues `fetch("file:///C:/Users/username/secret.txt")` or loads local documents in `<iframe>`.
- **Mitigation**:
  1. `file://` protocol is completely disabled in WebView2 settings.
  2. Content is loaded strictly via custom virtual host mapping: `https://kinetiwall.local/` mapped to the package directory.
  3. CORS and origin isolation prevent reading outside `kinetiwall.local`.
- **Residual Risk**: Negligible. Status: **MITIGATED IN DESIGN**.

#### KW-T-010: Raw Audio Eavesdropping via WASAPI Loopback Capture
- **STRIDE**: Information Disclosure | **Target**: `AST-05` (Audio Stream Privacy)
- **Attack Vector**: Malicious web wallpaper requests audio access to spy on microphone or audio playing through speakers (Zoom calls, private videos).
- **Mitigation**:
  1. Web wallpapers are denied microphone and audio capture permissions in WebView2 (`PermissionRequested` event handler returns `Deny`).
  2. The Core Controller captures WASAPI loopback, runs an in-memory FFT, downsamples into 16 or 32 normalized frequency floats, and delivers only the mathematical floats. Zero raw audio PCM samples are ever stored on disk or sent over IPC.
- **Residual Risk**: Low. Status: **MITIGATED IN DESIGN**.

#### KW-T-012: Giant Oversized IPC Payload Memory Exhaustion
- **STRIDE**: Denial of Service | **Target**: `AST-01` (Core Controller RAM)
- **Attack Vector**: Malicious or buggy client sends a 2 GB message length prefix, triggering out-of-memory allocation during read.
- **Mitigation**: Strict message framing check. The 4-byte length prefix is checked against `MAX_IPC_MESSAGE_SIZE` (4 MB). If prefix $> 4\text{ MB}$, the connection is severed immediately without allocating buffer memory.
- **Residual Risk**: Negligible. Status: **MITIGATED IN DESIGN**.

#### KW-T-013: Stale Player Instance Command Confusion (PID Reuse)
- **STRIDE**: Tampering | **Target**: `AST-06` (Process Handles)
- **Attack Vector**: A player crashes, Windows rapidly reallocates its PID to an unrelated user application, and the Core sends a termination or management signal to the wrong process.
- **Mitigation**: Core tracks processes by open Win32 `HANDLE` (which prevents Windows PID recycling until the handle is closed) AND a unique `Uuid` session token, never bare integer PIDs alone.
- **Residual Risk**: Negligible. Status: **MITIGATED IN DESIGN**.

#### KW-T-015: Config File Corrupt Overwrite via Sudden Power Loss
- **STRIDE**: Tampering / Denial of Service | **Target**: `AST-03` (Application Settings)
- **Attack Vector**: Power interruption mid-write truncates `config.json` to zero bytes.
- **Mitigation**: Win32 `MoveFileExW` atomic replacement + automatic restore from `config.json.bak` (`KW-DAT-007`).
- **Residual Risk**: Negligible. Status: **MITIGATED IN DESIGN**.

#### KW-T-019: Broken Player Process Becoming Orphaned Zombie
- **STRIDE**: Denial of Service | **Target**: System RAM / GPU Resources
- **Attack Vector**: Core Controller is terminated unexpectedly via Task Manager; child player processes continue running indefinitely behind desktop icons.
- **Mitigation**: All child processes are assigned to a Windows Job Object configured with `JOB_OBJECT_LIMIT_KILL_ON_JOB_CLOSE`. When the Core's job handle closes, the Windows kernel automatically terminates all child player processes instantly.
- **Residual Risk**: Negligible. Status: **MITIGATED IN DESIGN**.

