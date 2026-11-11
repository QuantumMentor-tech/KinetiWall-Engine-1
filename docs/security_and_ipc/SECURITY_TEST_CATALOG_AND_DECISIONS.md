# SECURITY ACCEPTANCE TEST CATALOG & DECISION REGISTER
## Document ID: KW-SEC-007 | Phase 2: System Architecture & Technical Decisions

---

### 1. Security Acceptance Test Catalog (Deliverable H)

This catalog defines 20 verifiable security test cases designed to validate the IPC boundaries, authorization controls, and sandboxing mechanisms established in Step 10.

> [!NOTE]
> In accordance with the documentation-only execution policy of Phase 2, zero test executions have been performed against live production binaries. **All tests are formally marked as `NOT RUN`.**

```
+------------------------------------------------------------------------------------------------------------------------+
|                                         SECURITY ACCEPTANCE TEST CATALOG SUMMARY                                       |
+---------+-------------------------------------------------------+-----------+----------+-------------------------------+
| TEST ID | SCENARIO DESCRIPTION                                  | THREAT ID | SEVERITY | TEST EXECUTION STATUS         |
+---------+-------------------------------------------------------+-----------+----------+-------------------------------+
| SAT-001 | Unauthorized Local Process Connects to Pipe Endpoint  | KW-T-002  | Critical | NOT RUN (Verification Planned)|
| SAT-002 | Unauthorized Command Requested by Gallery UI          | KW-T-002  | High     | NOT RUN (Verification Planned)|
| SAT-003 | Message Sent with Unsupported Protocol Version        | KW-T-011  | Medium   | NOT RUN (Verification Planned)|
| SAT-004 | Malformed JSON Payload Flooding                       | KW-T-011  | Medium   | NOT RUN (Verification Planned)|
| SAT-005 | Oversized IPC Payload (> 4 MB Length Prefix)          | KW-T-012  | High     | NOT RUN (Verification Planned)|
| SAT-006 | Invalid Identifier / Non-UUID String in Assignment   | KW-T-013  | Medium   | NOT RUN (Verification Planned)|
| SAT-007 | Player Process Attempts Direct Config Mutation        | KW-T-001  | Critical | NOT RUN (Verification Planned)|
| SAT-008 | Web Wallpaper Attempts Local File Access (file:///)   | KW-T-008  | High     | NOT RUN (Verification Planned)|
| SAT-009 | Web Wallpaper Attempts Native Command Execution       | KW-T-007  | Critical | NOT RUN (Verification Planned)|
| SAT-010 | Web Wallpaper Navigates Away to External Domain       | KW-T-009  | High     | NOT RUN (Verification Planned)|
| SAT-011 | Stale Command Delivered to Reused Player PID          | KW-T-013  | High     | NOT RUN (Verification Planned)|
| SAT-012 | Duplicate Non-Idempotent Request Replayed             | KW-T-020  | Medium   | NOT RUN (Verification Planned)|
| SAT-013 | IPC Disconnection During Wallpaper Assignment         | KW-T-015  | High     | NOT RUN (Verification Planned)|
| SAT-014 | Core Controller Hard Crash (Job Object Teardown)     | KW-T-019  | High     | NOT RUN (Verification Planned)|
| SAT-015 | Player Process Crash Mid-Playback                     | KW-T-019  | Medium   | NOT RUN (Verification Planned)|
| SAT-016 | Client Attempts Unauthorized Process Termination      | KW-T-001  | Critical | NOT RUN (Verification Planned)|
| SAT-017 | IPC Message Flooding Denial-of-Service                | KW-T-011  | Medium   | NOT RUN (Verification Planned)|
| SAT-018 | Inspection of Diagnostic Logs for PII / Audio Leaks   | KW-T-018  | High     | NOT RUN (Verification Planned)|
| SAT-019 | Ingestion of Malicious Zip Slip Archive               | KW-T-004  | Critical | NOT RUN (Verification Planned)|
| SAT-020 | Ingestion of Wallpaper Package Containing .exe/.bat   | KW-T-006  | Critical | NOT RUN (Verification Planned)|
+---------+-------------------------------------------------------+-----------+----------+-------------------------------+
```

---

### 2. Detailed Test Specifications

#### SAT-001: Unauthorized Local Process Connects to Pipe Endpoint
- **Threat Ref**: `KW-T-002` | **Severity**: Critical | **Status**: **NOT RUN**
- **Preconditions**: `kinetiwall-core.exe` running with `EP-CORE-UI` listening.
- **Stimulus**: A custom test executable running under the same user session connects to `\\.\pipe\kinetiwall-core-s1` and sends `{"payload_name": "REQ_SET_CONFIG"}`.
- **Expected Outcome**: The Core checks `GetNamedPipeClientProcessId`, finds the PID is not in its authenticated UI process registry, invokes `DisconnectNamedPipe`, logs `WARN_UNAUTHORIZED_CLIENT_PID`, and rejects the command.
- **Pass Condition**: Pipe disconnected; zero state changes in Core.
- **Fail Condition**: Command accepted or executed.

#### SAT-005: Oversized IPC Payload (> 4 MB Length Prefix)
- **Threat Ref**: `KW-T-012` | **Severity**: High | **Status**: **NOT RUN**
- **Preconditions**: Core listening on Named Pipe.
- **Stimulus**: Client sends 4-byte length prefix `0x00800000` (8 MB).
- **Expected Outcome**: Core reads the 4-byte prefix, detects length $> 4\text{ MB}$, aborts buffer allocation, and severs the pipe immediately.
- **Pass Condition**: Core process memory delta is $< 1\text{ KB}$; connection dropped.
- **Fail Condition**: Core allocates 8 MB or crashes with OOM.

#### SAT-008: Web Wallpaper Attempts Local File Access (`file:///`)
- **Threat Ref**: `KW-T-008` | **Severity**: High | **Status**: **NOT RUN**
- **Preconditions**: `kinetiwall-player-web.exe` running with test wallpaper loaded.
- **Stimulus**: Wallpaper JavaScript calls `fetch("file:///C:/Windows/System32/drivers/etc/hosts")`.
- **Expected Outcome**: WebView2 blocks the request with a DOM security error; virtual host mapping denies `file://` protocol.
- **Pass Condition**: Promise rejects with network/security error; file contents remain unread.
- **Fail Condition**: File contents successfully returned to JavaScript.

#### SAT-019: Ingestion of Malicious Zip Slip Archive
- **Threat Ref**: `KW-T-004` | **Severity**: Critical | **Status**: **NOT RUN**
- **Preconditions**: Core Controller running.
- **Stimulus**: Call `REQ_IMPORT_WALLPAPER` with a crafted zip containing `../../malicious.txt`.
- **Expected Outcome**: Staging validator identifies directory traversal component `..`, halts extraction, deletes the temporary staging directory, and returns `ERR_SECURITY_VIOLATION`.
- **Pass Condition**: `malicious.txt` is NOT written outside the staging directory.
- **Fail Condition**: File written to parent or root directories.

#### SAT-020: Ingestion of Wallpaper Package Containing `.exe` or `.dll`
- **Threat Ref**: `KW-T-006` | **Severity**: Critical | **Status**: **NOT RUN**
- **Preconditions**: Core Controller running.
- **Stimulus**: Import a `.kwp` archive containing `assets/payload.exe`.
- **Expected Outcome**: Archive scanner identifies `.exe` extension on blacklist, rejects import, logs `WARN_EXECUTABLE_REJECTED`.
- **Pass Condition**: Package rejected with `ERR_SECURITY_VIOLATION`; zero assets copied to library.
- **Fail Condition**: Package accepted or executable stored in wallpaper folder.

---

### 3. Open Security Decision Register (Deliverable I)

```
+--------------------------------------------------------------------------------------------------------------------------+
|                                           SECURITY DECISION REGISTER (STEP 10)                                           |
+-------------+-------------------------------------------------------+----------+-----------------------------------------+
| DECISION ID | TITLE / SUMMARY                                       | STATUS   | ARCHITECTURAL RATIONALE                 |
+-------------+-------------------------------------------------------+----------+-----------------------------------------+
| DEC-SEC-01  | Windows Named Pipes Transport with Length Framing     | APPROVED | Eliminates firewall prompts, sub-ms sync|
| DEC-SEC-02  | Explicit Win32 SDDL DACL Restricting to User SID      | APPROVED | Prevents cross-user pipe manipulation   |
| DEC-SEC-03  | Mandatory SECURITY_IDENTIFICATION Client SQOS         | APPROVED | Eliminates Win32 token impersonation    |
| DEC-SEC-04  | Kernel PID Authentication via GetNamedPipeClientPID   | APPROVED | Blocks spoofed local client commands    |
| DEC-SEC-05  | WebView2 Virtual Host Mapping (kinetiwall.local)      | APPROVED | Strict origin isolation without file:///|
| DEC-SEC-06  | In-Memory Downsampled Audio Spectrum (Zero Raw PCM)   | APPROVED | Prevents surveillance, preserves privacy|
| DEC-SEC-07  | Windows Job Object KILL_ON_JOB_CLOSE Lifecycle Binding| APPROVED | Guarantees zero zombie player processes |
| DEC-SEC-08  | Package Binary Whitelist & Blacklist Enforcement      | APPROVED | Blocks remote native code execution     |
+-------------+-------------------------------------------------------+----------+-----------------------------------------+
```

#### Detailed Decision Records:

- **DEC-SEC-01: Windows Named Pipes Transport**
  - *Context*: Selecting IPC mechanism between native Core, UI, and players.
  - *Decision*: Standardize on duplex Windows Named Pipes with 4-byte length-prefixed JSON frames.
  - *Consequences*: Sub-millisecond latency, zero network stack warnings, native Windows security controls.

- **DEC-SEC-02: Explicit SDDL DACLs for Named Pipes**
  - *Context*: Preventing unauthorized local processes from interacting with IPC.
  - *Decision*: Enforce explicit SDDL (`"D:P(A;;GA;;;PS)(A;;GA;;;OW)(A;;GA;;;BA)"`) on all created pipes.
  - *Consequences*: Only the current user's security token can open pipe handles; prevents unauthorized multi-user access.

- **DEC-SEC-03: Prohibition of Impersonation & Enforced SQOS**
  - *Context*: Mitigating Named Pipe impersonation attacks (`ImpersonateNamedPipeClient`).
  - *Decision*: Prohibit impersonation APIs codebase-wide; enforce `SECURITY_IDENTIFICATION` on all client connections.
  - *Consequences*: Prevents rogue clients or servers from elevating privileges via Windows tokens.

- **DEC-SEC-04: Client PID Verification via Win32 Kernel Query**
  - *Context*: Authenticating connecting processes without embedded secrets.
  - *Decision*: Query `GetNamedPipeClientProcessId` on accept and match against `LifecycleSupervisor` process table.
  - *Consequences*: Unregistered local processes are severed immediately before parsing any payload.

- **DEC-SEC-05: WebView2 Virtual Host Mapping Sandbox**
  - *Context*: Isolating untrusted interactive wallpaper web content.
  - *Decision*: Map assets to `https://kinetiwall.local/` via `SetVirtualHostNameToFolderMapping`; disable `file://` protocol.
  - *Consequences*: Content is bound by standard Web origin policies; local filesystem traversal is mathematically prevented.

- **DEC-SEC-06: In-Memory Downsampled Audio Spectrum Delivery**
  - *Context*: Audio visualizer wallpaper privacy.
  - *Decision*: Capture WASAPI loopback strictly inside Core, calculate 16-32 frequency buckets via FFT, stream only floats.
  - *Consequences*: Complete immunity against audio eavesdropping; zero raw PCM samples leave the Core.

- **DEC-SEC-07: Windows Job Object Lifecycle Binding**
  - *Context*: Preventing orphaned child player processes when Core crashes.
  - *Decision*: Assign all spawned player processes to a Windows Job Object with `JOB_OBJECT_LIMIT_KILL_ON_JOB_CLOSE`.
  - *Consequences*: OS automatically terminates all child processes when the Core controller process terminates.

- **DEC-SEC-08: Binary Blacklist for Package Ingestion**
  - *Context*: Defending against malicious wallpaper package archives.
  - *Decision*: Reject any package containing `.exe`, `.dll`, `.bat`, `.cmd`, `.ps1`, `.vbs`, `.reg`, or `.msi`.
  - *Consequences*: Prevents malicious wallpaper authors from packaging native malware.

---

### 4. Provisional Assumptions & Boundary Conditions

1. **WebView2 Runtime Availability**: Assumes Evergreen WebView2 Runtime is installed on target Windows systems (standard on Windows 11 and Windows 10 2004+). If absent, a fallback installer prompt is triggered.
2. **Standard User Token Execution**: KinetiWall is architected to run as a standard non-elevated user process. Elevation (Administrator / UAC) is neither required nor recommended.
3. **No External Executable Wallpapers in V1.0**: Third-party `.exe` wallpapers are excluded from V1.0 scope (`KW-NG-002`). All live wallpapers must be either video streams (mpv/MediaFoundation) or web packages (WebView2).

