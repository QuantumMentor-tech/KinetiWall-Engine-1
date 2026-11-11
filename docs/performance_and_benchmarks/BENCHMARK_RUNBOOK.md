# BENCHMARK EXECUTION RUNBOOK
## Document ID: KW-PRF-003 | Phase 2: System Architecture & Engineering Standards

---

### 1. Purpose & Reproducibility Guarantee

This runbook defines the exact, repeatable operating procedures for conducting performance benchmarks on KinetiWall. Any performance engineer following this protocol must be able to reproduce identical benchmark runs on reference hardware within a $\pm 5\%$ margin of measurement variance.

---

### 2. Reference Hardware & Test Environment Profiles

To ensure consistency across benchmark reports, tests must specify which standardized hardware tier was utilized:

```
+---------------------------------------------------------------------------------------------------------+
|                                      REFERENCE HARDWARE PROFILES                                        |
+--------------------------+------------------------------------+-----------------------------------------+
| HARDWARE SPECIFICATION   | TIER 1: HIGH-PERFORMANCE DESKTOP   | TIER 2: MAINSTREAM MOBILE (LAPTOP)      |
+--------------------------+------------------------------------+-----------------------------------------+
| Operating System         | Windows 11 Pro 23H2 / 24H2         | Windows 11 Home 23H2                    |
| Processor (CPU)          | AMD Ryzen 7 7800X3D or Intel i7-14700| Intel Core i5-1335U or AMD Ryzen 5 7530U|
| Cores / Threads          | 8 Cores / 16 Threads (or 20T)      | 10 Cores (2P+8E) / 12 Threads           |
| System Memory (RAM)      | 32 GB DDR5-6000                    | 16 GB LPDDR5-4800                       |
| Graphics Card (GPU)      | NVIDIA GeForce RTX 4070 (Discrete) | Intel Iris Xe / Radeon 610M (Integrated)|
| GPU Driver               | Latest Game Ready / Studio Driver  | OEM Verified WHQL Driver                |
| Primary Storage          | 1 TB NVMe PCIe 4.0 SSD             | 512 GB NVMe PCIe 3.0 SSD                |
| Displays                 | Dual: 1440p144Hz + 1080p60Hz       | Single: 1920x1080 @ 60Hz                |
| Power Configuration      | High Performance Power Plan (AC)   | Balanced Power Plan (Tested on AC & DC) |
+--------------------------+------------------------------------+-----------------------------------------+
```

---

### 3. Pre-Test Environmental Preparation (Clean State Protocol)

Before initiating any benchmark capture:
1. **Disable Competing Software**:
   - Verify no other wallpaper engines (Wallpaper Engine, Lively, AutoWall) are installed or running.
   - Terminate all third-party background utilities (Discord, Steam, MSI Afterburner, RGB controllers, cloud storage sync clients).
2. **Windows Maintenance Settle**:
   - Check Windows Update (`Settings` $\to$ `Windows Update`); ensure zero updates are downloading, installing, or pending restart.
   - Reboot the test machine.
   - Allow the system to settle in an idle state for **10 minutes** to ensure Windows startup indexing and superfetch tasks have completed.
3. **Display Settings**:
   - Verify native display resolution and scaling (100% or 125% DPI as specified in test record).
   - Ensure HDR is set to a consistent state (SDR recommended for standard baseline runs).

---

### 4. Tooling Setup & Instrumentation Configuration

#### 4.1 Windows Performance Monitor (PerfMon) Data Collector
Configure a user-defined Data Collector Set in Windows Performance Monitor (`perfmon.msc`):

```powershell
# Automated PowerShell script to create KinetiWall Benchmark Collector Set:
$counters = @(
    "\Process(kinetiwall*)\% Processor Time",
    "\Process(kinetiwall*)\Private Bytes",
    "\Process(kinetiwall*)\Working Set",
    "\Process(kinetiwall*)\Handle Count",
    "\Process(kinetiwall*)\Thread Count",
    "\Processor(_Total)\% Processor Time",
    "\GPU Engine(*)\Utilization Percentage"
)
logman create counter KinetiWall_Perf -si 00:00:01 -c $counters -f bld -o "C:\PerfLogs\KinetiWall_Run"
```

- **Sampling Interval**: Exactly 1.0 second (1,000 ms).
- **Log Format**: Binary Log Data (`.blg`), convertible to `.csv` via `relog.exe`.

#### 4.2 PresentMon Frame Presentation Instrumentation
For Direct3D presentation timing and dropped frame verification:
```powershell
PresentMon.exe -process_name kinetiwall-player-video.exe -output_file C:\PerfLogs\presentmon_output.csv -terminate_on_proc_exit
```

---

### 5. Step-by-Step Benchmark Execution Procedures

#### Protocol 1: S0 Clean Baseline Collection (300 Seconds)
1. Ensure KinetiWall is NOT running (`Get-Process kinetiwall*` returns null).
2. Start the PerfMon collector: `logman start KinetiWall_Perf`.
3. Leave the mouse and keyboard untouched for **300 seconds (5 minutes)**.
4. Stop the collector: `logman stop KinetiWall_Perf`.
5. Export average baseline CPU, memory, and handle usage.

#### Protocol 2: S1 Controller Idle Benchmark (300 Seconds)
1. Launch `kinetiwall-core.exe` (tray icon appears). Ensure no wallpapers are assigned.
2. Start the PerfMon collector.
3. Leave the machine untouched for **300 seconds**.
4. Stop the collector.
5. Calculate: $\text{Overhead}_{\text{Idle}} = \text{Mean}_{\text{S1}} - \text{Mean}_{\text{S0}}$.
6. Verify against `KW-MET-001` ($\le 15.0\text{ MB}$) and `KW-MET-002` ($\le 0.10\%$).

#### Protocol 3: S3 Active Video Playback Benchmark (600 Seconds)
1. Assign reference video `REF-VID-01` (1080p60 MP4) to the primary display.
2. Allow a **30-second warm-up window** (to exclude initial disk I/O, shader cache warming, and swapchain allocation).
3. Start PerfMon and PresentMon collectors simultaneously.
4. Record data for **600 seconds (10 minutes)**.
5. Stop collectors.
6. Extract average CPU%, peak private bytes, GPU decode engine utilization, and dropped frame count.

#### Protocol 4: S4 Fullscreen Pause & Resume Latency Benchmark
1. Launch `kinetiwall-player-video.exe` with active playback.
2. Launch a standardized full-screen DirectX reference app (e.g., Unigine Heaven or borderless 3D test harness).
3. Monitor the exact millisecond timestamps in `kinetiwall-core.log`:
   - $T_0$: `EVENT_SYSTEM_FOREGROUND` hook fired.
   - $T_1$: `EVT_PLAYBACK_STATE_CHANGED(Paused)` confirmed.
   - Calculate Pause Latency: $\Delta T_{\text{pause}} = T_1 - T_0$.
4. Minimize the fullscreen application.
   - $T_2$: Forefront window minimization detected.
   - $T_3$: First active D3D11 frame presented (`PresentMon` first timestamp).
   - Calculate Resume Latency: $\Delta T_{\text{resume}} = T_3 - T_2$.
5. Repeat 10 times to establish P95 and standard deviation.

---

### 6. Statistical Analysis & Outlier Rules

1. **Repetition Requirement**: Every benchmark scenario must be executed a minimum of **5 consecutive times**.
2. **Warm-up Exclusion**: The first 30 seconds of active playback are excluded from steady-state averages.
3. **Reported Statistics**:
   - **Mean ($\mu$)**: Arithmetic average.
   - **Median ($M$)**: 50th percentile.
   - **95th Percentile ($P_{95}$)**: Key metric for latency distributions.
   - **Standard Deviation ($\sigma$)**: Measure of dispersion.
   - **Coefficient of Variation ($CV = \sigma / \mu$)**: If $CV > 15\%$, investigate environmental noise.
4. **Outlier Discard Protocol**:
   - An individual run may ONLY be discarded if an explicit external OS event is verified in Event Viewer (e.g. `Event ID 10016 - DCOM Error`, Windows Defender scan start, or thermal throttling flag).
   - Deleting runs simply because they exceed target thresholds is strictly forbidden.

---

### 7. Teardown, Cleanup & System Restoration

Upon benchmark completion:
1. Terminate all test collectors: `logman stop KinetiWall_Perf`.
2. Exit KinetiWall via tray icon or `taskkill /IM kinetiwall-core.exe`.
3. Verify all player subprocesses are terminated:
   ```powershell
   Get-Process kinetiwall* | Format-Table Id, ProcessName, PrivateMemorySize64
   ```
4. Archive raw log files (`.blg`, `.csv`, `.log`) to the benchmark archive directory:
   `C:\PerfLogs\Archive\Run_{Date}_{RunID}\`.
5. Restore standard Windows power settings if modified during testing.

