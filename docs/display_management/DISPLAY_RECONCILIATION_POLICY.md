# Display Reconciliation & Recovery Policy
**Document ID:** `KW-DSP-004`  
**Deliverable:** D (Display Reconciliation and Recovery Strategy)  
**Parent Document:** [README.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/display_management/README.md)  
**Classification:** Core System Architecture / Operational Reliability Standard  

---

## 1. Event Ingress & Anti-Oscillation Debounce Architecture

Display configuration changes on Windows generate rapid bursts of operating system events. For example, hotplugging an HDMI cable or changing resolution can emit 4 to 8 consecutive `WM_DISPLAYCHANGE` messages within 300 ms as graphics drivers negotiate display timings and handshake EDIDs.

### 1.1 The Anti-Oscillation Pipeline

```
[Burst of WM_DISPLAYCHANGE Messages from Windows OS]
                        |
                        v
       1. Intercept Message in Core Hidden Window
                        |
                        v
       2. Reset 500 ms Monotonic Debounce Timer
   (Wait until graphics driver stabilizes; discard intermediate spikes)
                        |
                        v
       3. Increment TopologyGenerationId (+1)
                        |
                        v
       4. Execute Authoritative Snapshot Query
          - EnumDisplayMonitors + GetMonitorInfoW
          - SHA-256 EDID Resolution
                        |
                        v
       5. Reconcile Changes against Active Player Registry
```

* **Debounce Budget:** 500 ms hold-off timer. Guarantees that KinetiWall never executes expensive DirectX device recreations or player respawns during intermediate GPU driver mode switches.

---

## 2. Stale Reconciliation Protection via Generation IDs

To prevent race conditions where a slow or delayed reconciliation task overwrites a newer display configuration:

```rust
pub struct DisplayTopologyManager {
    /// Monotonically increasing generation counter
    current_generation: u64,
    /// Active authoritative snapshot
    active_snapshot: Option<TopologySnapshot>,
}

impl DisplayTopologyManager {
    pub fn handle_display_change(&mut self) {
        self.current_generation += 1;
        let expected_generation = self.current_generation;

        // Schedule async reconciliation after 500ms debounce
        tokio::spawn(async move {
            tokio::time::sleep(Duration::from_millis(500)).await;
            
            // Invariant Check: Verify this task is still the newest generation
            if !is_current_generation(expected_generation) {
                tracing::info!("Discarding stale display reconciliation (Gen {})", expected_generation);
                return;
            }
            
            execute_reconciliation(expected_generation).await;
        });
    }
}
```

---

## 3. Display Lifecycle Event Reconciliation Matrix

The following table defines the supervisor's deterministic response to every external display transition:

| OS Event / Transition | Ingress Signal | Core Action Sequence | Impact on Active Player | Expected Latency |
| :--- | :--- | :--- | :--- | :---: |
| **Resolution Hot-Change** | `WM_DISPLAYCHANGE` | Core detects updated bounds; queries new `rcMonitor`; sends `CMD_RESIZE_PLAYER(w, h)` via IPC. | Player resizes swapchain buffers via `ResizeBuffers`. Decoding continues uninterrupted. | $\le 100\text{ ms}$ |
| **Monitor Disconnected** | `WM_DISPLAYCHANGE` | Monitor missing from enumeration; Core marks slot `DISCONNECTED`; sends `CMD_TERMINATE_PLAYER`. | Player terminates cleanly within $250\text{ ms}$; VRAM and process handle freed. | $\le 300\text{ ms}$ |
| **Monitor Reconnected** | `WM_DISPLAYCHANGE` | Monitor appears in enumeration; matches EDID hash from `config.json`; spawns dedicated player process. | New player process spawned, assigned to Job Object, reparented into WorkerW. | $\le 450\text{ ms}$ |
| **Primary Display Swapped** | `WM_DISPLAYCHANGE` | Core recalculates virtual desktop origin (`SM_XVIRTUALSCREEN`); updates `workerw_bounds` for all displays. | Players update window positions via `SetWindowPos(SWP_NOZORDER)`. | $\le 150\text{ ms}$ |
| **Display Sleep (Power Off)**| `GUID_MONITOR_POWER_ON` (0) | Core signals all players `CMD_PAUSE_RENDERING`. | Swapchain `Present` halted; video decoder frozen; CPU drops to $< 0.1\%$. | $\le 20\text{ ms}$ |
| **Display Wake (Power On)** | `GUID_MONITOR_POWER_ON` (1) | Core signals all players `CMD_RESUME_RENDERING`. | Player presents last valid frame and resumes 60 FPS decoding. | $\le 50\text{ ms}$ |
| **Workstation Lock** | `WTS_SESSION_LOCK` | Core signals all players `CMD_PAUSE_RENDERING`. | Decoding paused; audio endpoints released. | $\le 20\text{ ms}$ |
| **Workstation Unlock** | `WTS_SESSION_UNLOCK` | Core signals all players `CMD_RESUME_RENDERING`. | Playback resumes instantly. | $\le 30\text{ ms}$ |

---

## 4. Preventing Duplicate Renderers & Process Leaks

A critical failure mode during display hotplugging is spawning multiple duplicate players for the same display. KinetiWall enforces three invariants:
1. **Single-Player Mutex per Display:** Core tracks active player processes in an internal registry keyed by `hardware_edid_hash`. Spawning a new player requires acquiring exclusive ownership of that monitor slot.
2. **Termination Before Recreation:** If a reconnect event triggers while an old player is still in `STOPPING` state, the spawn operation waits asynchronously for the old process handle to signal (`WAIT_OBJECT_0`) with an unyielding 250 ms timeout before launching the new player.
3. **Windows Job Object Containment:** All spawned player processes are bound to Core's Job Object (`DEC-PRF-08`), guaranteeing zero orphaned renderers if supervisor reconciliation fails.

---

## 5. Observable Error States & Degradation Policy

If display reconciliation fails (e.g. invalid Win32 coordinates, driver deadlocks, or ambiguous monitor identities), KinetiWall transitions to defined degraded states:

* **`STATE_DEGRADED_INVALID_GEOMETRY`:** Display reports negative dimensions or bounds exceeding 32k pixels. Core leaves native desktop wallpaper untouched and logs an error; no unanchored windows are spawned.
* **`STATE_DEGRADED_DRIVER_TIMEOUT`:** `EnumDisplayMonitors` hangs or times out. Core logs `WARN_DISPLAY_ENUM_TIMEOUT`, trips the circuit breaker, and pauses further reconciliation for 5 seconds.
* **`STATE_UNASSIGNED_FALLBACK`:** Newly connected monitor has no saved configuration. Core leaves the display idle or presents the user's default fallback wallpaper without overwriting existing assignments.

