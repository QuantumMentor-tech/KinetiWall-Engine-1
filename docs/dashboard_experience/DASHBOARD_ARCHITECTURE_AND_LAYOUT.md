# Dashboard Architecture & Layout Specification
**Document ID:** `KW-DSH-001`  
**Deliverable:** A (Dashboard Architecture and Layout)  
**Parent Document:** [README.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/dashboard_experience/README.md)  
**Classification:** Frontend Application Experience / Dashboard Layout Standard  

---

## 1. Dashboard Layout Blueprint & 4-Quadrant Visual Hierarchy

The Dashboard view (`src-ui/views/DashboardView.tsx`) renders inside the fluid `ContentViewport` established in Step 22. It organizes essential wallpaper controls and engine telemetry into four cohesive quadrants:

```
+-------------------------------------------------------------------------------------------------------+
| QUADRANT 1: Dashboard Header & Global State Bar (Height: ~64px)                                       |
|  - Title: "Dashboard Overview"                                                                        |
|  - Engine Runtime Status Badge: [ Active (60 FPS) ]  [ Battery: 85% (AC) ]  [ Displays: 2 Connected ]|
+-------------------------------------------------------------------------------------------------------+
| QUADRANT 2: Active Wallpaper Hero Card               | QUADRANT 3: Quick Action & Shortcut Grid       |
| (Width: 60% on Standard Desktop)                     | (Width: 40% on Standard Desktop)               |
|                                                      |                                                |
|  +------------------------------------------------+  |  +------------------------------------------+  |
|  | [16:9 Thumbnail Preview with Ambient Neon Glow]|  |  | [ ▶ / ⏸ ]  Pause / Resume Playback       |  |
|  +------------------------------------------------+  |  +------------------------------------------+  |
|  | Title: "Cyberpunk Metropolis 4K"               |  |  | [ 🔊 / 🔇 ]  Mute / Unmute Audio           |  |
|  | Author: "NeonForge"  | Type: Video (H.264)     |  +------------------------------------------+  |
|  | Target: Primary Display (\\.\DISPLAY1)          |  |  | [ 📁 ]       Open Wallpaper Library       |  |
|  | [ Customize Properties ]  [ Change Target ]    |  +------------------------------------------+  |
|  +------------------------------------------------+  |  | [ 🖥️ ]       Configure Multi-Monitor      |  |
|                                                      |  +------------------------------------------+  |
+-------------------------------------------------------------------------------------------------------+
| QUADRANT 4: Display Topology & System Health Cards (Bottom Band / Multi-Column Grid)                  |
|                                                                                                       |
|  +--------------------------------+ +--------------------------------+ +---------------------------+  |
|  | Primary Display: \\.\DISPLAY1  | | Secondary Display: \\.\DISPLAY2| | Engine Diagnostics        |  |
|  | 3840 x 2160 @ 60Hz (150% DPI)  | | 1920 x 1080 @ 144Hz (100% DPI) | | GPU VRAM: 312 MB / 8192 MB|  |
|  | Status: Rendering (mpv / D3D11)| | Status: Rendering (mpv / D3D11)| | CPU Usage: 0.4% (Normal)  |  |
|  +--------------------------------+ +--------------------------------+ +---------------------------+  |
+-------------------------------------------------------------------------------------------------------+
```

---

## 2. Component Composition & Reuse

The Dashboard is composed entirely of reusable primitives and composed patterns established in Step 21:

* **Header Status Badges:** Renders `Badge` components with semantic variant colors:
  - `emerald`: Engine operational (`STATE_RUNNING`, 60 FPS).
  - `amber`: Engine paused (`STATE_DISPLAY_INACTIVE` or `PauseReason::UserManual`).
  - `magenta`: Engine degraded (`STATE_LIFECYCLE_DEGRADED` or driver device loss).
* **Hero Panel Controls:** Reuses `Button` (`variant="neon"` for primary action, `variant="secondary"` for customization).
* **Quick Action Controls:** Uses `Button` components styled with large leading icons (`leftIcon`), clear action labels, and integrated async spinners (`isLoading`).
* **Confirmation Dialogs:** Destructive quick actions (e.g. "Stop All Playback") trigger `ConfirmationDialog` before dispatching IPC commands (`DEC-UI-03`).

---

## 3. Responsive Breakpoint Adaptation

The Dashboard dynamically reorganizes across the 4 desktop breakpoint tiers established in Step 22 (`RESPONSIVE_DESKTOP_LAYOUT.md`):

```
[COMPACT DESKTOP: 960px <= Width < 1200px]
  --> Stacked Single-Column Flow:
      - Quadrant 1: Full-width Header Bar
      - Quadrant 2: Full-width Active Wallpaper Hero Panel
      - Quadrant 3: 2x2 Grid of Quick Action Buttons below Hero
      - Quadrant 4: Horizontally scrollable Display Cards

[STANDARD DESKTOP: 1200px <= Width < 1600px]
  --> Two-Column Balanced Flow:
      - Left Column (60% width): Quadrant 2 (Hero Panel)
      - Right Column (40% width): Quadrant 3 (Quick Actions List)
      - Bottom Band: Quadrant 4 (Multi-Monitor Cards, 3 columns)

[WIDE & ULTRA-WIDE DESKTOP: Width >= 1600px]
  --> Three-Column Expansive Flow:
      - Left Column (45% width): Quadrant 2 (Expanded Hero Panel with full metadata)
      - Center Column (25% width): Quadrant 3 (Quick Actions & Settings shortcuts)
      - Right Column (30% width): Quadrant 4 (Real-time Display Topologies & VRAM gauges)
```

* **No Horizontal Overflow:** All containers enforce `max-w-full overflow-hidden`, preventing horizontal scrollbars even when high-resolution $3840 \times 2160$ monitor strings are displayed.

