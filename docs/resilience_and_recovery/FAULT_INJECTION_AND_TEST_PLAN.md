# FAULT-INJECTION & RECOVERY TEST PLAN
## Document ID: KW-RCV-004 | Phase 3: Observability, Diagnostics & Operational Reliability

---

### 1. Test Architecture & Safety Isolation Standards

Testing fault recovery requires injecting controlled software failures into active processes. To prevent destabilizing the host developer machine or corrupting operating system files, KinetiWall enforces three **Safety Isolation Rules**:
1. **Zero Global Operating System Disruption**: Tests MUST NOT kill `explorer.exe`, alter system-wide crash dump policies, or modify Windows system files.
2. **Mock Test Doubles for Environmental Faults**: Storage exhaustion and write locks are simulated via mocked filesystem layers or temporary RAM disks, never by filling the physical system drive.
3. **Isolated Test Sandboxes**: All tests execute inside temporary sandboxes (`%LOCALAPPDATA%\KinetiWall\test_sandboxes\{uuid}\`) cleaned up automatically upon test completion.

> [!NOTE]
> All test records in this plan are currently classified as **`NOT RUN (DESIGN PHASE)`**. Physical test execution will commence upon completion of the implementation phases (Phases 3–7).

---

### 2. Comprehensive Fault-Injection Matrix (`FINJ-001` .. `015`)

```
+-------------------------------------------------------------------------------------------------------------------------+
|                                           FAULT-INJECTION TEST MATRIX (15 SCENARIOS)                                    |
+----------+------------------------------------+-----------------------+--------------------+----------------------------+
| TEST ID  | TEST SCENARIO DESCRIPTION          | INJECTED FAULT        | RECOVERY METRIC    | PASS CRITERIA (OUTCOME)    |
+----------+------------------------------------+-----------------------+--------------------+----------------------------+
| FINJ-001 | Player Process Hard Crash Inject   | TerminateProcess      | Latency <= 1,000ms | Respawned with same UUID   |
| FINJ-002 | Crash Loop Circuit Breaker Trip    | 4 Rapid Crashes in 60s| Counter trips at 3 | Quarantined, desktop clean |
| FINJ-003 | IPC Response Timeout Injection     | Stall Named Pipe Read | Timeout = 2,000ms  | Reconnects pipe, resumes   |
| FINJ-004 | Renderer Init Handshake Timeout    | Delay first frame >3s | Timeout = 3,000ms  | Kills PID, retries once    |
| FINJ-005 | Direct3D11 Device Lost Injection   | DXGI_ERROR_DEVICE_REST| Recreate <= 500ms  | Rebuilds D3D11 swapchain   |
| FINJ-006 | Missing WebView2 Runtime Inject    | Unregister COM loader | Immediate Fallback | Disables web, video works  |
| FINJ-007 | WorkerW Hook Failure Simulation    | Hide WorkerW window   | Fallback <= 200ms  | Falls back to Progman hook |
| FINJ-008 | Media File Deletion During Play    | Delete active video   | Detection <= 100ms | Detaches surface, alert    |
| FINJ-009 | Display Disconnect During Video    | Detach secondary mon  | Reconcile <= 500ms | Primary mon unaffected     |
| FINJ-010 | User Cancel During Backoff Sleep   | Stop click during 2s  | Instant Cancel     | Timer aborts, zero respawn |
| FINJ-011 | Rapid Concurrent Wallpaper Switch  | 10 Switches in 100ms  | Serialized FIFO    | Latest timestamp applied   |
| FINJ-012 | Config Corrupt & .bak Restoration  | Write garbage to JSON | Boot time <= 850ms | Restores config.json.bak   |
| FINJ-013 | Logging Destination Write Lock     | Lock log directory    | Zero App Stalls    | Memory ring buffer active  |
| FINJ-014 | Core Controller Forced Kill        | Force-kill Core PID   | Cleanup <= 50ms    | Job Object kills players   |
| FINJ-015 | Child Process Shutdown Hang        | Infinite loop on exit | Force-kill = 250ms | Hard TerminateProcess      |
+----------+------------------------------------+-----------------------+--------------------+----------------------------+
```

---

### 3. Detailed Fault-Injection Test Specifications

#### `FINJ-001`: Player Process Hard Crash Injection (`TerminateProcess`)
- **Requirement Ref**: `KW-NF-012`, `KW-MET-013` | **Policy Ref**: `POL-05`
- **Preconditions**: `kinetiwall-core.exe` running with active 1080p60 video wallpaper on Monitor 1.
- **Injected Fault**: External test harness calls Win32 `OpenProcess(PROCESS_TERMINATE)` and `TerminateProcess(hPlayer, 0xC0000005)`.
- **Expected Outcome**:
  1. Core detects process exit via Tokio asynchronous child monitor within $< 50\text{ ms}$.
  2. Emits `PLAYER_CRASH_DETECTED` event with code `STATUS_ACCESS_VIOLATION`.
  3. Waits $1,000\text{ ms}$ exponential backoff delay.
  4. Respawns player subprocess with matching wallpaper UUID and re-attaches to `WorkerW`.
- **Pass Condition**: Total recovery time $\le 1,150\text{ ms}$; playback resumes seamlessly; desktop icons remain intact.
- **Status**: **NOT RUN (DESIGN PHASE)**

#### `FINJ-002`: Crash Loop Circuit Breaker Activation (4 Rapid Crashes in 60s)
- **Requirement Ref**: `KW-NF-012`, `KW-R-002` | **Policy Ref**: `POL-11`
- **Injected Fault**: Injected fault terminates the player subprocess immediately upon spawn, 4 consecutive times.
- **Expected Outcome**:
  1. Restarts 1, 2, and 3 execute with backoffs (1s, 2s, 4s).
  2. On 4th crash, `LifecycleSupervisor` trips the circuit breaker.
  3. Auto-restart permanently halts (`RECOVERY_EXHAUSTED`).
  4. Display cleanly resets to static Windows wallpaper.
  5. Action Center toast displayed to user with code `[KW-ERR-511]`.
- **Pass Condition**: Zero further processes spawned; Core CPU drops to $\le 0.10\%$; zero desktop lockups.
- **Status**: **NOT RUN (DESIGN PHASE)**

#### `FINJ-012`: Configuration File Corruption & Auto-Restore from `.bak`
- **Requirement Ref**: `KW-DAT-007`, `DEC-DATA-03` | **Policy Ref**: `POL-12`
- **Preconditions**: Verified `config.json` and `config.json.bak` exist in test sandbox.
- **Injected Fault**: Overwrite `config.json` with 0 bytes (simulating sudden power loss mid-write). Launch Core.
- **Expected Outcome**:
  1. JSON parser flags unexpected EOF.
  2. Bootstrapper loads `config.json.bak`, validates schema, and restores it over `config.json`.
  3. Emits `CONFIG_WRITE_COMPLETED` with recovery warning.
- **Pass Condition**: Core boots successfully in $\le 850\text{ ms}$; user settings fully preserved.
- **Status**: **NOT RUN (DESIGN PHASE)**

#### `FINJ-014`: Core Controller Forced Kill (Windows Job Object Teardown)
- **Requirement Ref**: `KW-NF-012`, `DEC-PRF-08` | **Policy Ref**: `POL-21`
- **Preconditions**: Core and two active player subprocesses rendering on dual monitors.
- **Injected Fault**: External process issues `TerminateProcess` against `kinetiwall-core.exe` (simulating sudden OS kill).
- **Expected Outcome**:
  1. The Core process handle closes abruptly.
  2. The Windows NT kernel automatically enforces `JOB_OBJECT_LIMIT_KILL_ON_JOB_CLOSE`.
  3. Both `kinetiwall-player-video.exe` instances are terminated by the kernel within $< 50\text{ ms}$.
- **Pass Condition**: Zero orphaned player processes left in Task Manager; zero desktop icon freezes.
- **Status**: **NOT RUN (DESIGN PHASE)**

---

### 4. Recovery Timing Metrics & Verification Standards

In alignment with **Step 12 (`PERFORMANCE_SPECIFICATION.md`)**, recovery timing must satisfy quantitative latency budgets:

```
+---------------------------------------------------------------------------------------------------------+
|                                      RECOVERY TIMING LATENCY BUDGETS                                    |
+------------------------------------------+-----------------------+--------------------------------------+
| RECOVERY LIFECYCLE PHASE                 | MAXIMUM ALLOWED TIME  | VERIFICATION METHOD                  |
+------------------------------------------+-----------------------+--------------------------------------+
| Subprocess Death Detection               | <= 50 ms              | Tokio child.wait() callback timer    |
| Recovery Policy Evaluation & Triage      | <= 5 ms               | In-memory state machine logic        |
| Exponential Backoff Timer (Attempt 1)    | Exactly 1,000 ms      | High-resolution timer (+- 50ms jitter|
| Fresh Player Process Spawn & Reparent    | <= 350 ms             | CreateProcessW to D3D11 Present      |
| Total Fault-to-Resume Latency (KW-MET-013| **<= 1,450 ms**       | End-to-end timestamp delta           |
+------------------------------------------+-----------------------+--------------------------------------+
```

---

### 5. Release-Blocking Reliability Conditions (Quality Exit Gates)

A build of KinetiWall MUST be blocked from release if any of the following reliability conditions occur during testing:

1. **Desktop Shell Freezing**: Any failure or recovery action causes Windows Explorer or desktop icons to freeze or become unclickable.
2. **Infinite Restart Loop**: Failure to trip circuit breaker after $\ge 3$ consecutive player crashes, leading to continuous process spawning.
3. **Ghost / Zombie Process Spawning**: Core termination leaves orphaned child processes running in the background.
4. **Configuration Data Loss**: An interrupted save or parsing failure results in loss of user preferences or deletion of the library.
5. **Security Sandbox Escape**: A player crash or error allows untrusted wallpaper script to execute native commands.
6. **User Control Lockout**: The user is unable to stop playback or close the application while a recovery cycle is pending.

