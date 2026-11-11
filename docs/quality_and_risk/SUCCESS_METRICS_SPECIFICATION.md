# KinetiWall — Success Metrics Specification
**Document ID:** `KW-QAR-002`  
**Deliverable:** Deliverable B — Success Metrics Specification  
**Phase:** 1 — Project Research, Requirements & Feasibility  
**Step:** 6 of 90 — Acceptance Criteria, Success Metrics & Risk Register  
**Status:** COMPLETE (Auditable Metric Specifications)  
**Authority:** Performance Engineering & Technical Program Management  
**Date:** 2026-11-10  

---

## 1. Specification Framework & Metric Categories

This document defines **30 formal success metrics** across 7 quality categories. Every metric is specified with 15 rigorous attributes to guarantee reproducibility across development milestones.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                       30 SUCCESS METRICS TAXONOMY                           │
├───────────────────────────────────┬─────────────────────────────────────────┤
│ Category                          │ Metric IDs                              │
├───────────────────────────────────┼─────────────────────────────────────────┤
│ 1. Performance & Resource Footprint│ KW-MET-001 to KW-MET-008                │
│ 2. Gaming & Occlusion Impact      │ KW-MET-009 to KW-MET-012                │
│ 3. Reliability & Recovery         │ KW-MET-013 to KW-MET-017                │
│ 4. Security & Privacy Assurance   │ KW-MET-018 to KW-MET-021                │
│ 5. Functional Correctness         │ KW-MET-022 to KW-MET-024                │
│ 6. Platform Compatibility Coverage│ KW-MET-025 to KW-MET-027                │
│ 7. Quality & Release Readiness    │ KW-MET-028 to KW-MET-030                │
└───────────────────────────────────┴─────────────────────────────────────────┘
```

---

## 2. Category 1: Performance & Resource Metrics

| Metric ID | Name | Definition | Units | Measurement Procedure | Baseline | Target Threshold | Target Authority | Review Milestone | Status |
|:---|:---|:---|:---:|:---|:---:|:---:|:---:|:---:|:---:|
| **`KW-MET-001`** | **Idle Core Daemon RAM** | Private working set of `kinetiwall-core.exe` running in background with UI closed. | Megabytes (MB) | VMMap / PerfMon sampling `Process(kinetiwall-core)\Private Bytes` every 5s for 1 hour. | Competitors: 70–180 MB | **≤15.0 MB** (Goal: 8–12 MB) | `APPROVED-DECISION` (`KW-NF-001`) | Phase 3 Core Foundation | `NOT RUN (PLANNED)` |
| **`KW-MET-002`** | **Idle Core Daemon CPU** | Average CPU utilization of core daemon when wallpaper is actively rendering or paused. | Percentage (%) | Windows Performance Recorder (WPR) sampled over 3,600s under normal desktop conditions. | Competitors: 0.3–1.5% | **≤0.10%** (Goal: 0.00%) | `APPROVED-DECISION` (`KW-NF-002`) | Phase 3 Core Foundation | `NOT RUN (PLANNED)` |
| **`KW-MET-003`** | **Ephemeral Gallery Closed RAM** | Resident memory footprint of `kinetiwall-ui.exe` after user closes the Gallery window. | Megabytes (MB) | Process table query for `kinetiwall-ui.exe` PID 5 seconds post-window close. | Competitors: 90 MB resident | **Exactly 0.0 MB** (Terminated) | `APPROVED-DECISION` (`KW-NG-001`) | Phase 5 UI Foundation | `NOT RUN (PLANNED)` |
| **`KW-MET-004`** | **1080p Video Playback CPU** | Sustained CPU usage of video player during 1080p 60fps H.264 playback. | Percentage (%) | WPR trace averaged over 600s uninterrupted playback on reference modern quad-core. | 2.5–5.0% (Software) | **≤1.00%** (D3D11VA HW) | `APPROVED-DECISION` (`KW-NF-003`) | Phase 4 Player Engine | `NOT RUN (PLANNED)` |
| **`KW-MET-005`** | **4K Video Playback CPU** | Sustained CPU usage of video player during 4K 60fps HEVC/AV1 playback. | Percentage (%) | WPR trace averaged over 600s on reference quad-core with hardware decoding. | 8.0–15.0% | **≤2.50%** (D3D11VA HW) | `APPROVED-DECISION` (`KW-NF-003`) | Phase 4 Player Engine | `NOT RUN (PLANNED)` |
| **`KW-MET-006`** | **Video Player VRAM Budget** | Dedicated GPU video memory allocated by player swapchain. | Megabytes (MB) | DXGI / GPU-Z dedicated video memory query on active player process. | 350–600 MB | **≤120 MB (1080p), ≤250 MB (4K)** | `PROPOSED TARGET` (`KW-NF-006`) | Phase 4 Player Engine | `NOT RUN (PLANNED)` |
| **`KW-MET-007`** | **Gapless Video Loop Delta** | Elapsed time delta between presentation of EOF frame and frame 0 at loop boundary. | Milliseconds (ms)| PresentMon frame-time timestamp capture at loop boundary across 20 cycles. | 50–200 ms (Stutter) | **≤16.6 ms** (1 frame @ 60Hz) | `APPROVED-DECISION` (`KW-F-011`) | Phase 4 Player Engine | `NOT RUN (PLANNED)` |
| **`KW-MET-008`** | **Audio Visualizer CPU Overhead** | CPU consumption of WASAPI loopback capture and 1024-point FFT calculation thread. | Percentage (%) | Dedicated thread profiler sampling FFT worker thread at 60 FPS over 300s. | 1.2–3.0% (Generic) | **≤0.30%** (SIMD-accelerated) | `PROPOSED TARGET` (`KW-NF-015`) | Phase 4 Audio Engine | `NOT RUN (PLANNED)` |

---

## 3. Category 2: Gaming & Occlusion Impact Metrics

| Metric ID | Name | Definition | Units | Measurement Procedure | Baseline | Target Threshold | Target Authority | Review Milestone | Status |
|:---|:---|:---|:---:|:---|:---:|:---:|:---:|:---:|:---:|
| **`KW-MET-009`** | **Fullscreen Pause Latency** | Elapsed time between fullscreen window foreground activation and swapchain presentation halt. | Milliseconds (ms)| High-precision software timestamping from `EVENT_SYSTEM_FOREGROUND` to player pause ACK. | 250–1000 ms (Polling)| **≤50.0 ms** | `APPROVED-DECISION` (`KW-NF-004`) | Phase 3 Event Engine | `NOT RUN (PLANNED)` |
| **`KW-MET-010`** | **Desktop Resume Latency** | Elapsed time between desktop exposure (minimizing windows) and first presented frame. | Milliseconds (ms)| High-speed screen recording (120 FPS) measuring frames from minimize to desktop wallpaper frame. | 50–150 ms | **≤16.6 ms** (Instant) | `PROPOSED TARGET` (`KW-F-028`) | Phase 3 Event Engine | `NOT RUN (PLANNED)` |
| **`KW-MET-011`** | **Gaming 99th % FPS Delta** | Variance in 99th percentile (1% low) game frame-times with KinetiWall paused vs closed. | Percentage (%) | CapFrameX 5-run benchmark on AAA 3D title comparing paused state to closed state. | 2.5–6.0% penalty | **<0.50% Variance** (Zero impact)| `APPROVED-DECISION` (`KW-NF-005`) | Phase 6 Performance Gate | `NOT RUN (PLANNED)` |
| **`KW-MET-012`** | **Paused State GPU Utilization** | GPU 3D and Video Decode load attributable to KinetiWall players while in PAUSED state. | Percentage (%) | NVAPI / AMD ADL GPU utilization query during active fullscreen game. | 1.0–4.0% | **Exactly 0.00%** | `APPROVED-DECISION` (`KW-F-030`) | Phase 3 Event Engine | `NOT RUN (PLANNED)` |

---

## 4. Category 3: Reliability & Recovery Metrics

| Metric ID | Name | Definition | Units | Measurement Procedure | Baseline | Target Threshold | Target Authority | Review Milestone | Status |
|:---|:---|:---|:---:|:---|:---:|:---:|:---:|:---:|:---:|
| **`KW-MET-013`** | **72-Hour Memory Drift Rate** | Linear change in total engine private working set over 72 hours of uninterrupted execution. | Megabytes (MB) | VMMap logging sampled every 60s for 259,200 seconds across 2 active display players. | 20–150 MB (Leaks) | **<1.0 MB Drift** (Zero leak) | `APPROVED-DECISION` (`KW-NF-007`) | Phase 6 Burn-In Gate | `NOT RUN (PLANNED)` |
| **`KW-MET-014`** | **Explorer Crash Recovery Time**| Elapsed time to re-attach wallpaper surface after `explorer.exe` restarts. | Milliseconds (ms)| Automated test script terminating Explorer and timing delta to `WorkerW` re-attachment. | 5,000–10,000 ms | **≤2,500 ms** | `APPROVED-DECISION` (`KW-F-003`) | Phase 3 Shell Engine | `NOT RUN (PLANNED)` |
| **`KW-MET-015`** | **Player Crash Recovery Rate** | Percentage of unexpected child player crashes cleanly restarted without daemon crash. | Percentage (%) | Automated fault injection killing player PID 50 consecutive times across test runs. | 75–85% | **100.0% Success** (50/50 runs) | `APPROVED-DECISION` (`KW-F-007`) | Phase 3 IPC Supervisor | `NOT RUN (PLANNED)` |
| **`KW-MET-016`** | **Orphan Process Incident Rate** | Number of lingering child processes remaining after abrupt termination of core controller. | Incidents | 50 consecutive hard kill aborts (`std::process::abort`) followed by process table audit. | 5–15 incidents | **Exactly 0 Incidents** (Job Obj) | `APPROVED-DECISION` (`KW-NF-010`) | Phase 3 Job Objects | `NOT RUN (PLANNED)` |
| **`KW-MET-017`** | **Display Change Recovery Rate**| Success rate of surviving monitor topology, resolution, and DPI changes without crash. | Percentage (%) | Automated script executing 30 display mode toggles and monitor hotplugs. | 80–90% | **100.0% Success** (30/30 runs) | `APPROVED-DECISION` (`KW-F-024`) | Phase 6 Topology Gate | `NOT RUN (PLANNED)` |

---

## 5. Category 4: Security & Privacy Metrics

| Metric ID | Name | Definition | Units | Measurement Procedure | Baseline | Target Threshold | Target Authority | Review Milestone | Status |
|:---|:---|:---|:---:|:---|:---:|:---:|:---:|:---:|:---:|
| **`KW-MET-018`** | **Outbound Telemetry Packets** | Total network packets transmitted to remote IP endpoints over 48 hours. | Packets | WireShark network packet capture on dedicated network interface over 172,800 seconds. | Competitors: Periodic pings | **Exactly 0 Packets** | `APPROVED-DECISION` (`KW-NG-002`) | Phase 7 Security Gate | `NOT RUN (PLANNED)` |
| **`KW-MET-019`** | **Disk Audio Write Bytes** | Total bytes of system audio data serialized or written to disk storage. | Bytes | Process Monitor file write audit filtering on audio capture thread ID over 24 hours. | 0 bytes | **Exactly 0 Bytes** (Volatile) | `APPROVED-DECISION` (`KW-NG-005`) | Phase 4 Audio Engine | `NOT RUN (PLANNED)` |
| **`KW-MET-020`** | **Web Sandbox CORS Escape Rate** | Successful unauthorized external network requests from web wallpaper scripts. | Escapes | Automated security penetration suite attempting 100 distinct cross-origin fetch/XHR calls. | 0 escapes | **Exactly 0 Escapes** (100% blocked) | `APPROVED-DECISION` (`KW-NF-016`) | Phase 4 Web Player | `NOT RUN (PLANNED)` |
| **`KW-MET-021`** | **Named Pipe Foreign Access Rate**| Successful IPC connections established from non-matching user security tokens. | Connections | Automated security harness attempting pipe connection from secondary non-elevated user account. | 0 connections | **Exactly 0 Connections** (DACL) | `APPROVED-DECISION` (`KW-NF-012`) | Phase 3 Security Gate | `NOT RUN (PLANNED)` |

---

## 6. Category 5: Functional Correctness Metrics

| Metric ID | Name | Definition | Units | Measurement Procedure | Baseline | Target Threshold | Target Authority | Review Milestone | Status |
|:---|:---|:---|:---:|:---|:---:|:---:|:---:|:---:|:---:|
| **`KW-MET-022`** | **V1.0 Functional Req Coverage** | Percentage of approved V1.0 functional requirements with passing automated tests. | Percentage (%) | Test execution matrix auditing `KW-F-001` through `KW-F-046` in `tests/acceptance/`. | 0% (Pre-impl) | **100.0% Pass Rate** | `APPROVED-DECISION` (`KW-SCP-002`) | Phase 7 Release Gate | `NOT RUN (PLANNED)` |
| **`KW-MET-023`** | **Wallpaper Switch Latency** | Elapsed time from user gallery click to active frame rendering on target display. | Milliseconds (ms)| Software timestamping from gallery IPC command to player first present call. | 800–2,000 ms | **≤500 ms** | `APPROVED-DECISION` (`KW-F-038`) | Phase 5 Gallery Engine | `NOT RUN (PLANNED)` |
| **`KW-MET-024`** | **Cold Startup Latency** | Time from process launch to presentation of initial desktop wallpaper frame. | Milliseconds (ms)| Cold boot timestamp log on reference NVMe SSD system. | 2,000–4,000 ms | **≤1,200 ms** | `APPROVED-DECISION` (`KW-NF-004`) | Phase 3 Core Engine | `NOT RUN (PLANNED)` |

---

## 7. Category 6: Platform Compatibility Metrics

| Metric ID | Name | Definition | Units | Measurement Procedure | Baseline | Target Threshold | Target Authority | Review Milestone | Status |
|:---|:---|:---|:---:|:---|:---:|:---:|:---:|:---:|:---:|
| **`KW-MET-025`** | **Tier A Platform Pass Rate** | Pass rate across 20 platform test cases (`CT-001`..`020`) on Windows 11 (24H2–22H2) and 10 (22H2). | Percentage (%) | Execution of test matrix defined in `docs/platform_support/TEST_MATRIX.md`. | 0% (Planned) | **100.0% Pass Rate** (0 failures)| `APPROVED-DECISION` (`DEC-PLT-002`) | Phase 6 Platform Gate | `NOT RUN (PLANNED)` |
| **`KW-MET-026`** | **Installer Package File Size** | Total standalone offline installer executable file size. | Megabytes (MB) | File size query of release build artifact `KinetiWall-Setup-v1.0.0-x64.exe`. | Competitors: 180–350 MB | **≤40.0 MB** | `PROPOSED TARGET` (`KW-NF-006`) | Phase 7 Packaging | `NOT RUN (PLANNED)` |
| **`KW-MET-027`** | **Air-Gapped Install Success** | Automated setup execution on offline Windows VM with zero network connectivity. | Pass / Fail | CI VM execution of installer with network adapter disabled; assert exit code 0. | Pass | **100% PASS** | `APPROVED-DECISION` (`KW-NG-003`) | Phase 7 Packaging | `NOT RUN (PLANNED)` |

---

## 8. Category 7: Quality & Release Governance Metrics

| Metric ID | Name | Definition | Units | Measurement Procedure | Baseline | Target Threshold | Target Authority | Review Milestone | Status |
|:---|:---|:---|:---:|:---|:---:|:---:|:---:|:---:|:---:|
| **`KW-MET-028`** | **Open P0 (Blocker) Defects** | Count of unresolved defects causing crashes, security escapes, or broken shell attachment. | Count | Issue tracker query for open issues marked Severity: Blocker or Priority: P0. | 0 | **Exactly 0 Defects** | `APPROVED-DECISION` (`KW-RQG-002`) | Phase 7 Release Gate | `NOT RUN (PLANNED)` |
| **`KW-MET-029`** | **Open P1 (Critical) Defects** | Count of unresolved functional defects affecting core video/web wallpaper playback. | Count | Issue tracker query for open issues marked Severity: Critical or Priority: P1. | 0 | **Exactly 0 Defects** | `APPROVED-DECISION` (`KW-RQG-002`) | Phase 7 Release Gate | `NOT RUN (PLANNED)` |
| **`KW-MET-030`** | **Core Unit Test Code Coverage** | Line coverage of pure-Rust business logic, IPC serialization, and manifest parsers. | Percentage (%) | `cargo-tarpaulin` code coverage report generated across workspace crates. | 0% | **≥75.0% Line Coverage** | `APPROVED-DECISION` (`KW-NF-027`) | Phase 6 QA Gate | `NOT RUN (PLANNED)` |

