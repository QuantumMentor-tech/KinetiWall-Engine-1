# STANDARDIZED PERFORMANCE RESULTS REPORT TEMPLATE
## Document ID: KW-PRF-004 | Phase 2: System Architecture & Engineering Standards

---

### Template Usage Instructions

> [!IMPORTANT]
> **Data Integrity Notice**:
> - Never leave blank cells or use unverified numbers.
> - Clearly distinguish unmeasured metrics awaiting physical execution (`[PENDING MEASUREMENT]`) from an observed, measured value of zero (`0.00`).
> - All statistical rows require a minimum sample of 5 verified repetitions following the protocol in [BENCHMARK_RUNBOOK.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/performance_and_benchmarks/BENCHMARK_RUNBOOK.md).

---

# KINETIWALL PERFORMANCE BENCHMARK REPORT
### Benchmark Run ID: `RUN-2026MMDD-XXXX`

---

### 1. Test Execution Metadata

| Parameter | Recorded Environment Details |
| :--- | :--- |
| **Execution Date & Time** | `YYYY-MM-DD HH:MM:SS UTC` |
| **Test Engineer / Lab ID** | `[Name or Lab Rig Identifier]` |
| **Git Commit Hash** | `[Full 40-char SHA e.g. af12e81...]` |
| **KinetiWall Build Type** | `[Release / Debug / Profile]` |
| **Operating System** | `Windows 11 Pro 64-bit (Version: 23H2, Build: 22631.XXXX)` |
| **Processor (CPU)** | `[e.g. AMD Ryzen 7 7800X3D (8C/16T @ 4.2GHz)]` |
| **System Memory (RAM)** | `[e.g. 32 GB DDR5-6000 CL30 (Available: XX GB)]` |
| **Graphics Card (GPU)** | `[e.g. NVIDIA GeForce RTX 4070 12GB]` |
| **GPU Driver Version** | `[e.g. 560.81 WHQL]` |
| **Primary Storage Device** | `[e.g. Samsung 990 Pro 2TB NVMe PCIe 4.0]` |
| **Display Configuration** | `[e.g. Display 1: 2560x1440 @ 144Hz (100% DPI)]` |
| **Power Profile** | `[High Performance / Balanced (AC Connected / DC Battery)]` |

---

### 2. Reference Workload & Asset Verification

| Workload ID | Description & Parameters | Asset Hash (SHA-256) |
| :--- | :--- | :--- |
| **REF-VID-01** | 1080p60 MP4 (H.264 / AVC), 12 Mbps, 60s Seamless Loop | `[e.g. e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855]` |
| **REF-WEB-01** | HTML5 Canvas Particle Visualizer (1,000 particles @ 60fps) | `[SHA-256]` |

---

### 3. Baseline Measurements (State S0: Application Absent)

*Measured over a 300-second quiet window prior to launching KinetiWall.*

| Metric Description | Metric Code | Run 1 | Run 2 | Run 3 | Run 4 | Run 5 | Mean ($\mu$) | StdDev ($\sigma$) |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **Total System CPU (%)** | `BASE-CPU` | `[PENDING]` | `[PENDING]` | `[PENDING]` | `[PENDING]` | `[PENDING]` | `[PENDING]` | `[PENDING]` |
| **Available Memory (MB)** | `BASE-RAM` | `[PENDING]` | `[PENDING]` | `[PENDING]` | `[PENDING]` | `[PENDING]` | `[PENDING]` | `[PENDING]` |
| **System Context Switches/s** | `BASE-CSW` | `[PENDING]` | `[PENDING]` | `[PENDING]` | `[PENDING]` | `[PENDING]` | `[PENDING]` | `[PENDING]` |

---

### 4. Application Workload Measurements & Baseline Comparison

#### 4.1 State S1: Controller Idle (Tray Icon Active, Zero Wallpapers)

| Performance Parameter | Target Spec | Observed Mean | Baseline ($\Delta$) | StdDev ($\sigma$) | P95 Value | Compliance Status |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **Core Private Bytes (MB)** | $\le 15.0\text{ MB}$ | `[PENDING]` | `+ [PENDING]` | `[PENDING]` | `[PENDING]` | `[PENDING]` |
| **Core CPU Utilization (%)** | $\le 0.10\%$ | `[PENDING]` | `+ [PENDING]` | `[PENDING]` | `[PENDING]` | `[PENDING]` |
| **Win32 OS Handle Count** | $\le 150$ | `[PENDING]` | `+ [PENDING]` | `[PENDING]` | `[PENDING]` | `[PENDING]` |
| **Wake-up Frequency (Hz)** | $\le 4.0\text{ Hz}$ | `[PENDING]` | `+ [PENDING]` | `[PENDING]` | `[PENDING]` | `[PENDING]` |

#### 4.2 State S3: Active 1080p60 Video Playback (`REF-VID-01`)

| Performance Parameter | Target Spec | Observed Mean | Baseline ($\Delta$) | StdDev ($\sigma$) | P95 Value | Compliance Status |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **Aggregate Process RAM (MB)**| $\le 65.0\text{ MB}$ | `[PENDING]` | `+ [PENDING]` | `[PENDING]` | `[PENDING]` | `[PENDING]` |
| **Aggregate Process CPU (%)**| $\le 2.00\%$ | `[PENDING]` | `+ [PENDING]` | `[PENDING]` | `[PENDING]` | `[PENDING]` |
| **Hardware Video Decode Util**| Active ($>0\%$) | `[PENDING]` | `N/A` | `[PENDING]` | `[PENDING]` | `[PENDING]` |
| **Dropped Frame Rate (%)** | $\le 0.10\%$ | `[PENDING]` | `N/A` | `[PENDING]` | `[PENDING]` | `[PENDING]` |

#### 4.3 State S4: Fullscreen Occlusion & Pause Latency

| Performance Parameter | Target Spec | Observed Mean | Median ($M$) | Min Value | Max Value | P95 Value | Compliance Status |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **Pause Reaction Latency (ms)**| $\le 100\text{ ms}$ | `[PENDING]` | `[PENDING]` | `[PENDING]` | `[PENDING]` | `[PENDING]` | `[PENDING]` |
| **Resume Present Latency (ms)** | $\le 150\text{ ms}$ | `[PENDING]` | `[PENDING]` | `[PENDING]` | `[PENDING]` | `[PENDING]` | `[PENDING]` |
| **Paused Total CPU Overhead (%)**| $\le 0.10\%$ | `[PENDING]` | `[PENDING]` | `[PENDING]` | `[PENDING]` | `[PENDING]` | `[PENDING]` |

#### 4.4 Lifecycle & UI Transitions

| Performance Parameter | Target Spec | Observed Mean | Baseline ($\Delta$) | StdDev ($\sigma$) | P95 Value | Compliance Status |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **Engine Cold Boot Time (ms)**| $\le 800\text{ ms}$ | `[PENDING]` | `N/A` | `[PENDING]` | `[PENDING]` | `[PENDING]` |
| **UI Closed Resident RAM (MB)**| **Exactly $0.0\text{ MB}$**| `[PENDING]` | `N/A` | `0.0` | `0.0` | `[PENDING]` |
| **24-Hour Memory Soak Delta** | $\le 2.0\text{ MB}$ | `[PENDING]` | `N/A` | `[PENDING]` | `[PENDING]` | `[PENDING]` |

---

### 5. Protocol Deviations & Environmental Anomalies

```
+---------------------------------------------------------------------------------------------------------+
| RECORDED DEVIATIONS OR UNEXPECTED SYSTEM EVENTS DURING RUN                                              |
| (If zero deviations occurred, record "NONE - STRICT PROTOCOL ADHERED TO")                               |
+---------------------------------------------------------------------------------------------------------+
| [Record any observed Windows Defender triggers, thermal throttling, or background network spikes]      |
+---------------------------------------------------------------------------------------------------------+
```

---

### 6. Evidence Artifacts & File Locations

- **PerfMon Counter Binary Log**: `C:\PerfLogs\RUN-2026MMDD-XXXX\KinetiWall_Perf.blg`
- **PresentMon Frame Trace CSV**: `C:\PerfLogs\RUN-2026MMDD-XXXX\presentmon_output.csv`
- **Application Diagnostic Log**: `C:\PerfLogs\RUN-2026MMDD-XXXX\kinetiwall_benchmark.log`
- **WPR ETW Trace Package**: `C:\PerfLogs\RUN-2026MMDD-XXXX\kinetiwall_etw.etl`

---

### 7. Final Verification Outcome & Sign-Off

- **Primary Benchmark Outcome**: `[PASS / FAIL / INCONCLUSIVE / PENDING PHYSICAL RUN]`
- **Blocking Issues Identified**: `[None / List Defect IDs]`
- **Performance Engineer Signature**: `[Pending Physical Execution]`
- **QA Sign-Off Date**: `[Pending Execution]`

