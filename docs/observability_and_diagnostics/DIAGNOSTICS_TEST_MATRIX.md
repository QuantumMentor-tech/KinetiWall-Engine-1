# LOGGING & DIAGNOSTICS TEST MATRIX
## Document ID: KW-OBS-004 | Phase 3: Observability, Diagnostics & Operational Reliability

---

### 1. Test Architecture & Verification Standards

This matrix specifies 20 verifiable diagnostic test cases covering schema correctness, multi-process event correlation, failure-injection resilience, automated privacy redaction, and logging performance budgets.

> [!NOTE]
> All test records in this matrix are currently classified as **`NOT RUN (DESIGN PHASE)`**. Physical test execution will commence upon completion of the implementation phases (Phases 3–7).

```
+---------------------------------------------------------------------------------------------------------+
|                                  DIAGNOSTICS TEST COVERAGE OVERVIEW                                     |
+---------+----------------------------------+-----------+--------------------+---------------------------+
| TIER    | TEST CATEGORY                    | ID RANGE  | TEST FOCUS         | CURRENT STATUS            |
+---------+----------------------------------+-----------+--------------------+---------------------------+
| Tier 1  | Schema & Basic Formatting        | DIAG-001..| JSON Schema, Timers| NOT RUN (DESIGN PHASE)    |
| Tier 2  | Multi-Process Correlation        | DIAG-006..| Correlation UUIDs  | NOT RUN (DESIGN PHASE)    |
| Tier 3  | Failure Injection & Resilience   | DIAG-009..| Disk Full, Perms   | NOT RUN (DESIGN PHASE)    |
| Tier 4  | Privacy & Redaction Engine       | DIAG-014..| Path/Token Scrub   | NOT RUN (DESIGN PHASE)    |
| Tier 5  | Diagnostic Export & Minidumps    | DIAG-018..| Zip Export, Dumps  | NOT RUN (DESIGN PHASE)    |
| Tier 6  | Performance & Resource Budget    | DIAG-020  | CPU & RAM Overhead | NOT RUN (DESIGN PHASE)    |
+---------+----------------------------------+-----------+--------------------+---------------------------+
```

---

### 2. Tier 1: Schema Correctness & Basic Formatting

#### `DIAG-001`: Structured JSON Schema Validity Verification
- **Requirement Ref**: `KW-F-043`, `CTR-06` | **Severity**: High | **Status**: **NOT RUN (DESIGN PHASE)**
- **Preconditions**: Test harness captures 500 emitted log lines across Core, UI, and Player processes.
- **Procedure**: Validate each captured line against JSON Schema V1 definition (`docs/observability_and_diagnostics/OBSERVABILITY_ARCHITECTURE.md`).
- **Pass Condition**: 100% of captured lines are valid single-line JSON. All mandatory fields present with correct types.
- **Fail Condition**: Any line with unescaped newlines, missing mandatory fields, or invalid types.

#### `DIAG-002`: Monotonic Timestamp & Elapsed Time Monotonicity
- **Requirement Ref**: `KW-F-043` | **Severity**: Medium | **Status**: **NOT RUN (DESIGN PHASE)**
- **Procedure**: Inspect `monotonic_elapsed_ms` across 1,000 sequentially emitted events while simultaneously adjusting the Windows system clock forward and backward by 1 hour.
- **Pass Condition**: `monotonic_elapsed_ms` values strictly increase monotonically without negative deltas.

#### `DIAG-003`: Severity Filtering Threshold Enforcement
- **Requirement Ref**: `KW-F-043` | **Severity**: Medium | **Status**: **NOT RUN (DESIGN PHASE)**
- **Procedure**: Set log level to `INFO`. Emit 100 `TRACE`, 100 `DEBUG`, and 100 `INFO` events.
- **Pass Condition**: Zero `TRACE` or `DEBUG` records written to disk file; all 100 `INFO` records written.

#### `DIAG-004`: Daily Log Rotation at Midnight UTC
- **Requirement Ref**: `KW-F-043`, `KW-DAT-001` | **Severity**: Medium | **Status**: **NOT RUN (DESIGN PHASE)**
- **Procedure**: Simulate system clock passing `23:59:59 UTC` to `00:00:00 UTC`. Emit an event.
- **Pass Condition**: A new log file `kinetiwall_core_{NewDate}.log` is created. Previous log handle is cleanly closed.

#### `DIAG-005`: Size-Based Log Rollover at 10 MB
- **Requirement Ref**: `KW-F-043` | **Severity**: Medium | **Status**: **NOT RUN (DESIGN PHASE)**
- **Procedure**: Write continuous diagnostic events until log reaches 10.0 MB.
- **Pass Condition**: Current file is renamed to `kinetiwall_core_{Date}.1.log`; fresh file created; zero lost records.

---

### 3. Tier 2: Multi-Process Correlation

#### `DIAG-006`: End-to-End Correlation ID Propagation
- **Requirement Ref**: `KW-F-043`, `ADR-0002` | **Severity**: High | **Status**: **NOT RUN (DESIGN PHASE)**
- **Procedure**: Gallery UI triggers `REQ_ASSIGN_WALLPAPER(UUID)`. Trace the generated `correlation_id` through the Core Controller state change and into the spawned Video Player subprocess `CMD_INIT_PLAYER`.
- **Pass Condition**: All log records generated across the 3 processes for this assignment share the exact same `correlation_id`.

#### `DIAG-007`: Unique Process Instance ID Assignment on Respawn
- **Requirement Ref**: `KW-NF-012`, `ADR-0014` | **Severity**: High | **Status**: **NOT RUN (DESIGN PHASE)**
- **Procedure**: Force-kill a player subprocess. Supervisor restarts the player.
- **Pass Condition**: The respawned player emits a new `process_instance_id` distinct from the crashed process instance, preventing diagnostic ambiguity.

#### `DIAG-008`: Disconnected Process Log Assembly
- **Requirement Ref**: `KW-F-043` | **Severity**: Medium | **Status**: **NOT RUN (DESIGN PHASE)**
- **Procedure**: Inspect `%LOCALAPPDATA%\KinetiWall\logs\` after a multi-monitor session.
- **Pass Condition**: `kinetiwall_core_{date}.log`, `player_video_1.log`, and `player_video_2.log` exist independently with cleanly separated records.

---

### 4. Tier 3: Failure Injection & Resilience

#### `DIAG-009`: Log Directory Write Permission Denied Handling
- **Requirement Ref**: `KW-NF-012` | **Severity**: Critical | **Status**: **NOT RUN (DESIGN PHASE)**
- **Procedure**: Set `%LOCALAPPDATA%\KinetiWall\logs\` permissions to Read-Only (`icacls logs /deny Everyone:(W)`). Launch Core and play wallpaper.
- **Pass Condition**: Core detects write failure; falls back to in-memory ring buffer; wallpaper renders without crash or stall.

#### `DIAG-010`: Simulated Disk-Full Condition Fallback
- **Requirement Ref**: `KW-NF-012` | **Severity**: Critical | **Status**: **NOT RUN (DESIGN PHASE)**
- **Procedure**: Inject mock `std::io::ErrorKind::StorageFull` into the logging file writer thread.
- **Pass Condition**: Writer switches to in-memory buffer; drops oldest events when buffer hits 256 entries; Core does not crash.

#### `DIAG-011`: Corrupted Previous Log File Recovery
- **Requirement Ref**: `KW-DAT-007` | **Severity**: High | **Status**: **NOT RUN (DESIGN PHASE)**
- **Procedure**: Pre-populate `logs/kinetiwall_core_{date}.log` with non-UTF8 random binary garbage. Launch Core.
- **Pass Condition**: Core appends a fresh recovery delimiter line and resumes normal logging; does not crash or erase corrupted file.

#### `DIAG-012`: High-Volume Log Event Queue Saturation / Drop Test
- **Requirement Ref**: `KW-NF-001`, `KW-MET-001` | **Severity**: High | **Status**: **NOT RUN (DESIGN PHASE)**
- **Procedure**: Flood the logging channel with 50,000 synthetic events in 100 ms.
- **Pass Condition**: Crossbeam channel bounds memory to $\le 2.0\text{ MB}$; lowest-severity events are dropped; main thread never blocks.

#### `DIAG-013`: Abrupt Process Termination & 250ms Shutdown Flush
- **Requirement Ref**: `KW-NF-006` | **Severity**: Medium | **Status**: **NOT RUN (DESIGN PHASE)**
- **Procedure**: Initiate application exit with 500 events pending in logging queue.
- **Pass Condition**: Writer flushes buffer within 250 ms; process terminates cleanly without hanging.

---

### 5. Tier 4: Privacy & Redaction Engine Verification

#### `DIAG-014`: User Profile Username Redaction Verification
- **Requirement Ref**: `KW-NF-019`, `KW-R-018` | **Severity**: Critical | **Status**: **NOT RUN (DESIGN PHASE)**
- **Procedure**: Emit an error message containing `C:\Users\SecretUser123\AppData\Local\KinetiWall\wallpapers\test.mp4`.
- **Pass Condition**: Log line serialized to disk reads `C:\Users\[USER]\AppData\Local\KinetiWall\wallpapers\test.mp4`. The string `SecretUser123` is completely absent.

#### `DIAG-015`: Bearer Token & API Key Redaction Verification
- **Requirement Ref**: `KW-NF-019` | **Severity**: Critical | **Status**: **NOT RUN (DESIGN PHASE)**
- **Procedure**: Emit a log message containing `Authorization: Bearer abcd1234efgh5678ijkl9012mnop3456`.
- **Pass Condition**: Log line serialized to disk reads `Authorization: Bearer=[REDACTED_TOKEN]`.

#### `DIAG-016`: Window Title Anonymization in Fullscreen Detection
- **Requirement Ref**: `KW-NF-019`, `ADR-0005` | **Severity**: High | **Status**: **NOT RUN (DESIGN PHASE)**
- **Procedure**: Focus a window titled `Personal_Bank_Statement.pdf - Adobe Acrobat`.
- **Pass Condition**: Log record context records `foreground_process: "AcroRd32.exe"`, `window_state: "Maximized"`. The document title string is completely omitted.

#### `DIAG-017`: Log Injection Neutralization (Newline / Control Byte Escaping)
- **Requirement Ref**: `KW-SEC-004`, `KW-T-018` | **Severity**: High | **Status**: **NOT RUN (DESIGN PHASE)**
- **Procedure**: Ingest wallpaper package with title `Evil\r\n{"severity":"FATAL","message":"Fake injected log"}`.
- **Pass Condition**: Single-line log integrity preserved; raw newlines are escaped as `\r\n`.

---

### 6. Tier 5: Diagnostic Export & Minidumps

#### `DIAG-018`: Local Diagnostic Zip Export Integrity & Manifest
- **Requirement Ref**: `KW-F-043` | **Severity**: High | **Status**: **NOT RUN (DESIGN PHASE)**
- **Procedure**: Trigger "Export Diagnostics" in UI.
- **Pass Condition**: Generates a valid `.zip` archive on Desktop containing sanitized logs, sanitized config, and `manifest.json`. Zero network packets emitted.

#### `DIAG-019`: Minidump Quota Enforcement & Zero Network Transmission
- **Requirement Ref**: `KW-NF-019` | **Severity**: High | **Status**: **NOT RUN (DESIGN PHASE)**
- **Procedure**: Enable crash dumps; inject 5 consecutive test crashes.
- **Pass Condition**: Directory `%LOCALAPPDATA%\KinetiWall\crashes\` retains exactly 3 dumps ($\le 45\text{ MB}$); oldest 2 are pruned; zero network connections.

---

### 7. Tier 6: Performance Overhead Verification

#### `DIAG-020`: Logging CPU & Memory Overhead Verification
- **Requirement Ref**: `KW-NF-001`, `KW-NF-002` | **Severity**: High | **Status**: **NOT RUN (DESIGN PHASE)**
- **Procedure**: Measure CPU and memory of Core Controller over 1 hour of sustained 1080p60 playback with default logging active.
- **Pass Condition**: Logging subsystem adds $\le 0.05\%$ CPU overhead; logging ring buffer consumes $\le 2.0\text{ MB}$ private memory.

