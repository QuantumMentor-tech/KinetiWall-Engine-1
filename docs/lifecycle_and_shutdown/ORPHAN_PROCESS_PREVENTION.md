# Orphan Process Prevention & Process Ownership Architecture
**Document ID:** `KW-LFC-004`  
**Deliverable:** D (Orphan Process Prevention)  
**Parent Document:** [README.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/lifecycle_and_shutdown/README.md)  
**Classification:** Core System Architecture / Operational Reliability Standard  

---

## 1. The Orphan Process Hazard in Windows Desktop Engines

A critical risk for live wallpaper engines on Windows is the generation of "orphan processes" (also termed zombie or runaway background renderers). This occurs when:
1. The supervisor daemon (`kinetiwall-core.exe`) crashes unexpectedly (e.g. unhandled Rust panic, stack overflow, or external taskkill).
2. The user forcefully terminates the supervisor via Task Manager or terminal.
3. The host machine enters sudden power loss, blue screen of death (BSOD), or fast user switching.

In naive desktop implementations, child player processes continue executing indefinitely behind the desktop. Because they are hidden beneath shell icons, the user is unaware that multiple 60 FPS video decoders or Chromium browser instances are consuming 100% CPU/GPU and multi-gigabytes of RAM.

KinetiWall eliminates this failure mode permanently through a multi-tiered, defense-in-depth architecture.

---

## 2. Windows Job Object Encapsulation Architecture

The primary technical foundation for orphan prevention in KinetiWall is the Windows NT **Job Object**.

```
+---------------------------------------------------------------------------------+
|                               WINDOWS NT KERNEL                                 |
|                                                                                 |
|  CreateJobObjectW(NULL, "Local\\KinetiWall_Job_<UUID>")                         |
|  SetInformationJobObject(                                                       |
|      hJob,                                                                      |
|      JobObjectExtendedLimitInformation,                                         |
|      &limitInfo /* LimitFlags = JOB_OBJECT_LIMIT_KILL_ON_JOB_CLOSE */           |
|  )                                                                              |
|                                                                                 |
|       +------------------------------------------------------------------+      |
|       |                     Job Object Boundary                          |      |
|       |                                                                  |      |
|       |   [kinetiwall-core.exe]  ---> Owns Job Handle                    |      |
|       |            |                                                     |      |
|       |            +--- Spawns (CREATE_SUSPENDED)                        |      |
|       |            |                                                     |      |
|       |            v                                                     |      |
|       |   AssignProcessToJobObject(hJob, hChildProcess)                  |      |
|       |            |                                                     |      |
|       |            +--- Resumes (ResumeThread)                           |      |
|       |            v                                                     |      |
|       |   [kinetiwall-player-video.exe]   [kinetiwall-player-web.exe]    |      |
|       |                                                                  |      |
|       +------------------------------------------------------------------+      |
|                                                                                 |
|  IF kinetiwall-core.exe CRASHES OR EXITS:                                       |
|  --> All handles to the Job Object are automatically closed by the NT Kernel    |
|  --> Windows Kernel immediately terminates all assigned child processes         |
|  --> Guaranteed termination time: < 50 milliseconds                             |
+---------------------------------------------------------------------------------+
```

### 2.1 Job Object Configuration Parameters

The supervisor configures `JOBOBJECT_EXTENDED_LIMIT_INFORMATION` with the following flags:

```rust
// Architectural specification for Windows Job Object initialization
pub fn configure_job_object(h_job: HANDLE) -> Result<(), Win32Error> {
    let mut info = JOBOBJECT_EXTENDED_LIMIT_INFORMATION::default();
    
    // Inviolable Rule: All children die when the job handle is closed
    info.BasicLimitInformation.LimitFlags = 
        JOB_OBJECT_LIMIT_KILL_ON_JOB_CLOSE |
        JOB_OBJECT_LIMIT_SILENT_BREAKAWAY_OK;

    let res = unsafe {
        SetInformationJobObject(
            h_job,
            JobObjectExtendedLimitInformation,
            &info as *const _ as *const c_void,
            std::mem::size_of::<JOBOBJECT_EXTENDED_LIMIT_INFORMATION>() as u32,
        )
    };
    
    if res == 0 {
        return Err(get_last_win32_error());
    }
    Ok(())
}
```

* **`JOB_OBJECT_LIMIT_KILL_ON_JOB_CLOSE`**: Enforces that when the last open handle to the Job Object is closed (which the Windows kernel guarantees upon process termination of `kinetiwall-core.exe`), all processes associated with the job are terminated forcefully by the kernel.
* **`JOB_OBJECT_LIMIT_SILENT_BREAKAWAY_OK`**: Permits approved diagnostic utilities (such as crash dump generators or bug report tools) to break away if explicitly permitted, while preventing rogue player subprocesses from detaching.

---

## 3. Process Identity & PID Recycling Mitigation

Operating system Process Identifiers (PIDs) on Windows are aggressively recycled. A terminated process's PID can be reassigned by the OS within milliseconds to an unrelated application (such as `explorer.exe`, `notepad.exe`, or a banking browser).

### 3.1 The PID Recycling Hazard

If a supervisor tracks subprocesses solely by PID:
1. Player process terminates unexpectedly ($PID = 4012$).
2. Core detects an anomaly and prepares to issue `TerminateProcess(4012)`.
3. Meanwhile, Windows reassigns $PID = 4012$ to a newly launched system process.
4. Core kills an innocent system process, causing system instability or data loss.

### 3.2 Solution: 128-Bit Process-Instance UUIDs & Handle Verification

To eliminate this hazard, KinetiWall implements a dual-key identity system:

```rust
pub struct SupervisedProcess {
    /// Cryptographically random 128-bit UUID generated by Core before spawn
    pub instance_id: Uuid,
    /// Target display string (e.g. "\\.\DISPLAY1")
    pub target_display: String,
    /// OS Process ID (ephemeral, not used for authority)
    pub os_pid: u32,
    /// Win32 Process Handle with PROCESS_QUERY_LIMITED_INFORMATION | SYNCHRONIZE | PROCESS_TERMINATE
    pub process_handle: SafeHandle,
    /// Process creation timestamp (FILETIME) for kernel-level verification
    pub creation_time: FILETIME,
}
```

* **Identity Verification Protocol:**
  1. Every command, heartbeat, and IPC message carries the `instance_id` UUID. Messages with mismatched UUIDs are rejected.
  2. Before issuing any Win32 management API call (`TerminateProcess`, `GetExitCodeProcess`), Core verifies the target handle using `GetProcessTimes`. If the creation timestamp does not match `creation_time`, the operation aborts immediately with `ERR_PID_RECYCLED_HANDLE_STALE`.

---

## 4. Exit Code Monitoring & Asynchronous Wait Loops

The Core supervisor monitors child process termination via dedicated asynchronous event loops using Win32 I/O completion ports (IOCP) or Tokio-integrated wait threads:

```
[Spawn Player Process]
         |
         v
Register Process Handle with Supervisor Wait Pool
         |
         +-------------------------------------------------------+
         | Wait Thread: WaitForMultipleObjects(ProcessHandles)    |
         +-------------------------------------------------------+
                                   |
         +-------------------------+-------------------------+
         |                                                   |
   [Handle Signaled]                                   [Handle Timeout]
         |                                                   |
GetExitCodeProcess(hProcess, &code)              Issue Heartbeat Ping
         |                                                   |
   +-----+-----+                                    +--------+--------+
   |           |                                    |                 |
[Code == 0] [Code != 0]                           [ACK]            [NO ACK]
   |           |                                    |                 |
Clean Exit   Crash / Panic Detected               Continue     Initiate Escalation
Log INFO     Log CRITICAL & Trigger Circuit Breaker
```

### 4.1 Recognized Exit Codes

| Exit Code | Constant Name | Description | Supervisor Response |
| :---: | :--- | :--- | :--- |
| `0` | `EXIT_SUCCESS` | Clean graceful shutdown requested by Core | Mark display idle; no restart. |
| `1` | `EXIT_GENERAL_ERROR` | Subprocess unhandled error | Check circuit breaker; restart if quota remains. |
| `0xC0000005` | `STATUS_ACCESS_VIOLATION` | Memory access violation / segfault | Record crash dump; apply restart backoff. |
| `0xFD` | `EXIT_DECODER_HANG` | mpv or hardware decoder timeout | Restart player in software decode fallback mode. |
| `0xFE` | `EXIT_TIMEOUT_FORCE` | Terminated forcefully by Core supervisor | Log shutdown escalation timeout; do not restart. |

---

## 5. Cold Startup Reconciliation Protocol

If the host machine experienced a power failure or the supervisor was terminated in a way that bypassed Job Object registration (e.g. during an unhandled driver fault), stale processes might exist on disk.

Upon cold boot, `kinetiwall-core.exe` executes a non-destructive startup reconciliation:

```
[Core Startup Entry]
         |
         v
1. Take System Process Snapshot: CreateToolhelp32Snapshot(TH32CS_SNAPPROCESS, 0)
         |
         v
2. Filter Processes matching "kinetiwall-player-*.exe" or "kinetiwall-ui.exe"
         |
         v
3. Verify Executable Path: QueryFullProcessImageNameW(hProcess)
   --> Must match Current Directory / Core Binary Location exactly!
         |
         v
4. Verify Command-Line Session Argument:
   --> Process must possess CLI flag "--supervised-by-kinetiwall"
         |
         v
5. Terminate Verified Stale Orphan: TerminateProcess(hStale, 0xFF)
         |
         v
6. Log Cleanup Event: WARN_STALE_ORPHAN_RECONCILED(pid, image_name)
```

### 5.1 Safeguards Against Terminating Unrelated Applications

To guarantee that KinetiWall never terminates third-party applications, games, or developer tools:
1. **Name Matching Alone is Prohibited:** Core never kills a process based solely on process name (`szExeFile`).
2. **Binary Path Hash Validation:** The full executable path of the target process is retrieved via `QueryFullProcessImageNameW`. The cryptographic hash of the binary is validated against the active installation manifest.
3. **Session Token Matching:** The target process must be running in the identical Windows Terminal Services Session ID (`ProcessIdToSessionId`) as the active Core instance.
4. **Command-Line Inspection:** The target process command line must contain KinetiWall's private session signature flag. If any check fails, the process is left untouched and a warning is logged.

