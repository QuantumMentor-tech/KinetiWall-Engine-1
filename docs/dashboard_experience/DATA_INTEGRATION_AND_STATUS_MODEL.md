# Dashboard Data Integration & Status Model
**Document ID:** `KW-DSH-002`  
**Deliverable:** B (Data Integration and Status Model)  
**Parent Document:** [README.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/dashboard_experience/README.md)  
**Classification:** Frontend Data Contracts / IPC Bridge Standard  

---

## 1. Truthful Data Invariant & Authoritative Source Mapping

In strict accordance with `Principle A`, the Dashboard presents **only verified operational data** queried directly from `kinetiwall-core.exe`. Fabricated or simulated values are strictly prohibited.

### 1.1 Data Source Authority Mapping

| Dashboard Metric / Information | Authoritative Backend Source | Query / Event Mechanism | Fallback Behavior if Unavailable |
| :--- | :--- | :--- | :--- |
| **Engine Runtime State** | `SystemLifecycleState` (Core) | `REQ_GET_DASHBOARD_STATE` | Displays "Connecting to Core Daemon..." |
| **Active Wallpaper Title & Type** | `config.json` (`display_profiles`) | `REQ_GET_DASHBOARD_STATE` | If unassigned: renders "No Active Wallpaper" |
| **Thumbnail Preview URL** | Local Asset Cache (`appDataDir`) | Virtual Host (`https://kinetiwall.local/thumbnails/`) | Renders placeholder SVG pattern card |
| **Connected Displays & DPI** | `TopologySnapshot` (Core) | `REQ_GET_DISPLAYS` / `topology-changed` | Renders "Detecting connected monitors..." |
| **Real-Time Presentation FPS** | Subprocess Telemetry Heartbeat | `telemetry-heartbeat` event push | Omitted; never displays fake "60 FPS" |
| **VRAM / GPU Utilization** | DXGI / D3D11 Query in Subprocess | `telemetry-heartbeat` event push | Displays "Hardware metrics disabled" |
| **Active Pause Reasons** | `PauseReasonSet` (Core Bitmask) | `REQ_GET_DASHBOARD_STATE` | Displays "Active" if empty; lists active reasons |

---

## 2. IPC Data Transfer Object (`DashboardBootstrapDTO`)

Upon mounting, the Dashboard acquires its initial state via a single, consolidated IPC call:

```typescript
export interface DashboardBootstrapDTO {
  engineState: 'running' | 'paused' | 'degraded' | 'stopped';
  activePauseReasons: string[]; // e.g. ["UserManual", "BatterySaving"]
  activeWallpaper: {
    id: string;
    title: string;
    author: string;
    wallpaperType: 'video' | 'web';
    thumbnailPath: string;
    targetDisplayName: string;
    durationSeconds?: number;
    volume: number;
    isMuted: boolean;
  } | null;
  connectedDisplaysCount: number;
  totalLibraryItemsCount: number;
  systemMetrics: {
    fps: number | null;
    vramUsedMb: number | null;
    vramTotalMb: number | null;
    cpuPercent: number | null;
  } | null;
}
```

### 2.1 Passive Event-Driven Synchronization

Following the initial bootstrap query, the Dashboard subscribes to three passive Tauri event bridges:
1. **`ipc://playback-state-changed`:** Pushed whenever Core pauses, resumes, or stops a player. Updates the Play/Pause button label and active status badge.
2. **`ipc://topology-changed`:** Pushed when a monitor is hotplugged or resolution changes. Updates the Display Topology summary card.
3. **`ipc://telemetry-heartbeat`:** Pushed at 1 Hz intervals containing true frame rate and VRAM usage. If telemetry is disabled in settings, this event stream is silent.

---

## 3. Quick Action Execution Contracts

Quick action triggers on the Dashboard dispatch commands to Core through strongly typed service adapters (`src-ui/services/dashboardService.ts`):

```typescript
export async function togglePlaybackPause(isCurrentlyPaused: boolean): Promise<void> {
  const command = isCurrentlyPaused ? 'resume_all_playback' : 'pause_all_playback';
  await invoke(command);
}

export async function toggleGlobalMute(isCurrentlyMuted: boolean): Promise<void> {
  await invoke('set_global_mute', { mute: !isCurrentlyMuted });
}

export function navigateToLibrary(): void {
  useRouterStore.getState().navigate('library');
}

export function navigateToDisplays(): void {
  useRouterStore.getState().navigate('displays');
}
```

* **Ingress Protection (`DEC-UI-04`):** Buttons enter `isLoading = true` upon click. If the user clicks "Pause", the button shows a loading spinner until Core emits `playback-state-changed`, preventing duplicate IPC commands.

