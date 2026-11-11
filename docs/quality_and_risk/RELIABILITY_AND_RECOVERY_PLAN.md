# KinetiWall — Reliability and Recovery Verification Plan
**Document ID:** `KW-QAR-004`  
**Deliverable:** Deliverable D — Reliability and Recovery Verification Plan  
**Phase:** 1 — Project Research, Requirements & Feasibility  
**Step:** 6 of 90 — Acceptance Criteria, Success Metrics & Risk Register  
**Status:** COMPLETE (Methodology & Recovery Protocol Baseline)  
**Authority:** Senior QA Architect & Reliability Engineering  
**Date:** 2026-11-10  

---

## 1. Reliability Philosophy & Core Objectives

Desktop wallpaper engines run 24 hours a day, 7 days a week. Unlike ordinary productivity applications, a live wallpaper engine cannot afford memory leaks, unhandled exceptions, zombie processes, or shell attachment corruption.

**The Golden Reliability Invariant:** A failure in an individual wallpaper player, third-party video codec, or web script must **never crash the core controller, degrade operating system responsiveness, or leave the Windows desktop in an unrecoverable state**.

---

## 2. Controlled Fault-Injection Protocol Suite

The table below catalogs the 8 mandatory fault-injection stress scenarios designed to validate system resilience:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                 CONTROLLED FAULT INJECTION SCENARIO SUITE                   │
├─────────┬─────────────────────────────┬──────────────┬──────────────────────┤
│ Test ID │ Fault Injected              │ Test Volume  │ Required Pass Rate   │
├─────────┼─────────────────────────────┼──────────────┼──────────────────────┤
│ REL-01  │ Explorer Shell Crash        │ 30 Iterations│ 100% Recovery ≤2.5s  │
│ REL-02  │ Child Player Process Kill   │ 50 Iterations│ 100% Restart ≤1.5s   │
│ REL-03  │ Controller Hard Crash       │ 50 Iterations│ 0 Orphan Processes   │
│ REL-04  │ Named Pipe IPC Disconnect   │ 100 Cycles   │ Re-connect ≤500ms    │
│ REL-05  │ Display Hotplug & Topology  │ 30 Events    │ 100% Re-Anchor       │
│ REL-06  │ S3 Sleep & Modern Standby   │ 25 Cycles    │ 100% Wake Resume     │
│ REL-07  │ Corrupt Media & Manifest    │ 50 Bad Files │ 0 Daemon Crashes     │
│ REL-08  │ 72-Hour Continuous Burn-In  │ 72 Hours     │ <1 MB Memory Drift   │
└─────────┴─────────────────────────────┴──────────────┴──────────────────────┤
```

---

## 3. Detailed Fault-Injection Specifications

### `REL-01`: Explorer Shell Crash & Re-Attachment Stress Test
- **Failure Mechanism:** Forceful termination of `explorer.exe` via Win32 `TerminateProcess` or `taskkill /F /IM explorer.exe`.
- **Preconditions:** KinetiWall running with dual video wallpapers across 2 monitors.
- **Execution Protocol:** 
  1. Trigger Explorer termination.
  2. Windows shell restarts automatically.
  3. Core daemon receives `TaskbarCreated` registered window message.
  4. Core re-executes message `0x052C` to `Progman` and re-anchors wallpaper child surfaces.
- **Sample Size / Denominator:** 30 consecutive iterations (1 iteration every 60 seconds).
- **Observable Evidence:** High-speed screen recording verifying desktop restoration; log file with timestamp delta.
- **Pass Criteria:** 30 out of 30 iterations (100%) re-establish desktop attachment within ≤2,500ms; zero core daemon crashes; zero desktop icon freezes.
- **Fail Criteria:** Core daemon crashes; wallpaper window becomes a detached floating window; icons become unclickable; desktop turns permanently black.
- **Status:** `NOT RUN (PLANNED)`

### `REL-02`: Player Subprocess Crash & Bounded Auto-Restart
- **Failure Mechanism:** Simulating GPU driver crashes or codec segmentation faults by killing `kinetiwall-player-video.exe` via `TerminateProcess`.
- **Preconditions:** Video wallpaper actively rendering on primary monitor.
- **Execution Protocol:**
  1. Inject hard process kill into child player.
  2. Core daemon detects child process handle signaled state via async thread pool wait.
  3. Core logs diagnostic event, waits exponential backoff interval (100ms, 500ms, 1500ms), and spawns replacement player process.
  4. Test rapid crash loop: kill replacement player 3 consecutive times within 60 seconds.
- **Sample Size / Denominator:** 50 independent crash iterations; 10 rapid crash loop iterations.
- **Pass Criteria:** 
  - Single crash recovers and resumes video playback within ≤1,500ms (100% success rate).
  - Triple crash halts auto-restart, reverts monitor to static wallpaper, and logs actionable user notification.
- **Fail Criteria:** Infinite crash restart loop consumes 100% CPU; core daemon crashes; desktop left black.
- **Status:** `NOT RUN (PLANNED)`

### `REL-03`: Controller Abrupt Termination & Zero-Orphan Verification
- **Failure Mechanism:** Controller abort (`std::process::abort()`) simulating an unhandled panic or hard kill.
- **Preconditions:** Core controller running with 3 child processes (1 video player, 1 web player, 1 audio visualizer helper) bound to Windows Job Object.
- **Execution Protocol:**
  1. Terminate `kinetiwall-core.exe` abruptly.
  2. PowerShell script instantly queries process table for all child PIDs every 10ms for 2,000ms.
- **Sample Size / Denominator:** 50 consecutive abort iterations.
- **Observable Evidence:** Process table snapshot log confirming zero child PIDs exist 100ms post-controller death.
- **Pass Criteria:** Exactly 0 child player processes remain running across all 50 test iterations (100% cleanup via `JOB_OBJECT_LIMIT_KILL_ON_JOB_CLOSE`).
- **Fail Criteria:** Even 1 child player remains executing as an orphan background process.
- **Status:** `NOT RUN (PLANNED)`

### `REL-04`: Named Pipe Disconnection & Re-Sync Resilience
- **Failure Mechanism:** Abruptly closing the Named Pipe IPC handle from the UI or player client while mid-transaction.
- **Preconditions:** Gallery UI communicating with core daemon over `\\.\pipe\kinetiwall-ipc-{SessionId}`.
- **Execution Protocol:**
  1. Send partial JSON-RPC payload and sever pipe connection abruptly.
  2. Verify core daemon flushes buffer, closes invalid handle, and re-enters listening state (`ConnectNamedPipe`).
  3. Client reconnects and transmits valid command.
- **Sample Size / Denominator:** 100 randomized disconnect cycles.
- **Pass Criteria:** 100% of reconnect attempts succeed within ≤500ms; zero daemon deadlocks; zero memory growth.
- **Fail Criteria:** Daemon hangs waiting for unread bytes; pipe enters broken state requiring service restart.
- **Status:** `NOT RUN (PLANNED)`

### `REL-05`: Multi-Monitor Hotplug & Topology Change Stress Test
- **Failure Mechanism:** Rapidly changing display topology: unplugging secondary HDMI/DP display, toggling resolutions (4K to 1080p), and changing DPI scaling.
- **Preconditions:** Dual-monitor setup with mixed DPI (Display 1: 150%, Display 2: 100%).
- **Execution Protocol:**
  1. Execute synthetic display disconnect via Display Driver Uninstaller (DDU) test tool or physical KVM switch.
  2. Core receives `WM_DISPLAYCHANGE` and re-queries `EnumDisplayMonitors`.
  3. Active player on disconnected display is gracefully destroyed; surviving display player re-anchors.
  4. Display reconnected; player re-spawns and resumes playback.
- **Sample Size / Denominator:** 30 display mode toggles.
- **Pass Criteria:** 100% clean recovery; zero coordinate drift; zero crashes; re-anchor completes in ≤1,000ms.
- **Fail Criteria:** Windows misaligned across screen; engine crashes on `WM_DISPLAYCHANGE`.
- **Status:** `NOT RUN (PLANNED)`

### `REL-06`: S3 Sleep, Modern Standby & Workstation Lock Recovery
- **Failure Mechanism:** Windows power transitions: entering S3 sleep, modern standby (Connected Standby), and workstation lock (`Win+L`).
- **Preconditions:** Active live wallpapers rendering across all displays.
- **Execution Protocol:**
  1. Broadcast `WM_WTSSESSION_CHANGE(WTS_SESSION_LOCK)`.
  2. Verify players pause swapchain and release audio capture within 100ms.
  3. System enters sleep; system wakes from sleep.
  4. Broadcast `WTS_SESSION_UNLOCK`.
  5. Verify players re-initialize D3D11 swapchains and resume playback within ≤500ms.
- **Sample Size / Denominator:** 25 sleep/wake and lock/unlock cycles.
- **Pass Criteria:** 100% seamless resume; zero black screen artifacts; zero audio device lockups.
- **Fail Criteria:** Wallpaper fails to resume; player crashes upon sleep wake; black desktop.
- **Status:** `NOT RUN (PLANNED)`

---

## 4. Protocol 8: 72-Hour Continuous Burn-In Stress Test (`REL-08`)

### 4.1 Objective
Empirically prove long-term longevity, memory stability, and leak-free execution over 72 hours of uninterrupted execution.

### 4.2 Test Configuration
- **Hardware:** Dedicated Tier 2 Mid-Range PC (Intel i5-12400, RTX 3060, 16 GB DDR4).
- **Displays:** 2 Physical Monitors (Monitor 1: 1440p 144Hz, Monitor 2: 1080p 60Hz).
- **Workloads:** 
  - Monitor 1: Seamlessly looping 4K 60fps HEVC video wallpaper.
  - Monitor 2: Interactive WebGL audio-reactive canvas wallpaper.
- **Duration:** 72 continuous hours (259,200 seconds).

### 4.3 Monitored Failure Indicators
1. **Private Working Set Drift:** Total memory growth from hour 1 to hour 72 must be **<1.0 MB** (VMMap leak analysis).
2. **Handle Leaks:** Process handle count must remain completely flat (variance ≤5 handles).
3. **GDI Object Leaks:** GDI objects allocated by daemon must not exceed 50 objects throughout run.
4. **Unhandled Panics:** Exactly 0 crashes, restarts, or unhandled exceptions in Windows Event Log.

---

## 5. Reliability Metric Calculation Standards

$$\text{Recovery Rate (\%)} = \left( \frac{\text{Successful Automated Recoveries}}{\text{Total Controlled Faults Injected}} \right) \times 100$$

Every reliability report must explicitly state the **numerator, denominator, and exact injection method**. Vague claims of "crash-proof" execution are strictly prohibited.

