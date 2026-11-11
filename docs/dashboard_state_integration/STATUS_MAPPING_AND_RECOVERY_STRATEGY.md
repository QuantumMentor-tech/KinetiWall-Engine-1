# Dashboard Status Mapping and Recovery Strategy
**Document ID:** `KW-DSI-003`  
**Phase:** 4 (UI/UX Design System & Dashboard)  
**Step:** 24 of 90  
**Status:** Approved Architectural Specification  
**Classification:** Frontend Application Architecture / Error Containment & Resilience Strategy  

---

## 1. Executive Summary & Purpose

This specification defines the authoritative status mapping taxonomy, runtime schema validation boundary, bounded retry policies, and degraded presentation fallbacks for the KinetiWall Dashboard (`src-ui/views/DashboardView.tsx`).

The core architectural requirement is **fault isolation**: unexpected states, network/IPC corruptions, hardware display disconnects, or player crashes must be contained gracefully without causing white-screen crashes, unhandled React rejections, or cascading failures across the desktop environment.

---

## 2. Centralized Status Mapping Matrix

KinetiWall rigorously decouples three distinct operational dimensions:
1. **Engine Daemon Health:** The operational status of the background supervisor process (`kinetiwall-core.exe`).
2. **Wallpaper Playback State:** The active rendering state of the wallpaper content.
3. **Display Hardware Topology:** The physical and virtual state of connected Windows displays.

| Operational Dimension | Status Code | UI Visual Indicator | Semantic Meaning & System Behavior |
| :--- | :--- | :--- | :--- |
| **Engine Health** | `Optimal` | Solid Green Pill (`#00FF88`) | Daemon running normally; IPC responsive; resource limits within threshold. |
| | `Degraded` | Pulsing Amber Pill (`#FFB800`) | Hardware acceleration fallen back to software, or high memory threshold reached. Engine operational. |
| | `Critical` | Solid Red Pill (`#FF3B30`) | Graphics adapter reset detected; IPC reconnecting; automatic player restart in progress. |
| | `Unresponsive` | Strikethrough Muted Pill | IPC heartbeat lost for $> 5000\text{ ms}$; reconnect attempts active. |
| **Playback State** | `Playing` | Active Glow & Play Icon | Wallpaper actively rendering frames at target refresh rate ($60\text{ FPS}$). |
| | `Paused` | Amber Pause Badge | Render loop paused via user action or system configuration. Audio muted. |
| | `Occluded` | Blue Shield Badge (`#00F0FF`) | Wallpaper occluded by maximized/fullscreen window; rendering halted to save battery/GPU (`KW-F-004`). |
| | `Stopped` | Gray Stop Badge | No active wallpaper loaded or playback explicitly terminated. |
| | `Error` | Red Alert Triangle | Wallpaper player crashed or asset missing; fallback wallpaper rendered. |
| **Display Slot Status**| `Active` | Bright Monitor Frame | Display active and rendering assigned wallpaper profile. |
| | `Paused` | Dimmed Monitor Frame | Display active but rendering paused. |
| | `Detached` | Dashed Outline with Warning | Monitor was physically disconnected; slot preserved for hot-reconnect (`KW-F-002`). |
| | `Faulted` | Red Hash Fill | DirectX surface recreation failed on this specific screen; other displays unaffected. |

---

## 3. Runtime Payload Schema Validation Boundary

To guarantee that malformed or corrupted IPC payloads cannot cause runtime crashes in `kinetiwall-ui.exe`, all incoming data passing through Tauri invoke or event channels is validated at the transport gate using `Zod` schemas:

```typescript
import { z } from 'zod';

export const DisplaySlotSchema = z.object({
  displayId: z.string().min(1),
  displayName: z.string(),
  isPrimary: z.boolean(),
  resolution: z.object({
    width: z.number().int().positive(),
    height: z.number().int().positive(),
  }),
  scaleFactor: z.number().positive(),
  refreshRateHz: z.number().positive(),
  slotStatus: z.enum(['Active', 'Paused', 'Disconnected', 'Faulted']),
  assignedWallpaperTitle: z.string().nullable(),
});

export const DashboardBootstrapSchema = z.object({
  schemaVersion: z.number().int().min(1),
  timestamp: z.number(),
  engine: z.object({
    daemonStatus: z.enum(['Connected', 'Degraded', 'Disconnected']),
    engineHealth: z.enum(['Optimal', 'Degraded', 'Critical']),
    activeProfileName: z.string(),
    uptimeSeconds: z.number().nonnegative(),
    version: z.string(),
  }),
  playback: z.object({
    globalState: z.enum(['Playing', 'Paused', 'Occluded', 'Stopped']),
    masterMuted: z.boolean(),
    masterVolume: z.number().min(0).max(1),
  }),
  activeWallpaper: z.object({
    id: z.string().nullable(),
    title: z.string(),
    artist: z.string(),
    engineType: z.enum(['Video', 'Web', 'Shader', 'Static', 'None']),
    thumbnailUri: z.string().nullable(),
    renderResolution: z.string(),
  }),
  displays: z.array(DisplaySlotSchema),
  telemetry: z.object({
    currentFps: z.number().nonnegative(),
    cpuUtilizationPct: z.number().min(0).max(100),
    gpuUtilizationPct: z.number().min(0).max(100),
    vramAllocatedMb: z.number().nonnegative(),
  }),
});
```

### 3.1 Validation Failure Handling
If payload parsing fails:
1. The raw payload is trapped before reaching the Zustand store.
2. A structured validation error is logged with details of mismatched fields.
3. The store retains its last known good state or falls back to safe default values.
4. An error boundary flag is set, showing a subtle warning banner: *"Diagnostics notice: IPC telemetry schema mismatch detected."*

---

## 4. Bounded Retry & Circuit Breaker Architecture

When the UI detects that the Core daemon is unreachable or failing commands, it engages a bounded circuit breaker to prevent resource exhaustion:

```
  +------------------+
  |  State: CLOSED   |  (Normal operation; commands flow freely)
  +------------------+
           |
           | 3 consecutive IPC timeouts / errors
           v
  +------------------+
  |   State: OPEN    |  (Fast-fail all commands; show 'Connecting...' banner;
  +------------------+   arm exponential backoff timer: 500ms -> 1000ms -> 2000ms)
           |
           | After backoff timeout expires
           v
  +------------------+
  | State: HALF-OPEN |  (Dispatch single ping test REQ_PING)
  +------------------+
      /          \
     / (Success)  \ (Failure)
    v              v
 CLOSED           OPEN (Max 3 retries; then transition to Permanent Disconnected)
```

### 4.1 Reconnection Parameters
* **Initial Probe Delay:** $500\text{ ms}$
* **Backoff Multiplier:** $2.0\times$
* **Maximum Probe Delay:** $2000\text{ ms}$
* **Max Consecutive Attempts:** $3$
* **Terminal State:** If 3 consecutive probes fail, the UI stops automatic attempts and displays a persistent recovery card:
  - Title: *"Engine Daemon Not Responding"*
  - Body: *"KinetiWall Core background process is temporarily unavailable."*
  - Primary Action: `"Restart Engine"` (`CMD_RESTART_CORE_DAEMON`)
  - Secondary Action: `"View Diagnostic Logs"`

---

## 5. Non-Destructive Degraded Fallback Modes

To maintain user confidence and prevent visual degradation, KinetiWall implements graceful fallbacks for specific failure modes:

### 5.1 Faulted Display Slot Fallback
* **Failure Trigger:** Display device fails DirectX surface allocation or crashes `kinetiwall-player.exe`.
* **Behavior:** Only the faulted display card shows an alert state (`slotStatus = 'Faulted'`). Unaffected monitors continue rendering without disruption. The user is offered a one-click `"Reinitialize Display"` button.

### 5.2 Missing or Corrupt Thumbnail Fallback
* **Failure Trigger:** `thumbnailUri` points to a missing file or an invalid image format.
* **Behavior:** The Hero panel seamlessly renders a dark geometric gradient background with an SVG engine icon (`VideoIcon`, `WebIcon`, `ShaderIcon`) matching the wallpaper type. The application never renders broken image icons or missing asset placeholders.

### 5.3 High DPI & Re-scaling Disruption
* **Failure Trigger:** User moves dashboard window across monitors with different DPI scales ($100\%$ vs $200\%$).
* **Behavior:** Zustand listens to `WM_DPICHANGED` via Tauri and triggers a smooth CSS transform re-scale without dropping active action states or clearing form inputs.

