# Dashboard Experience Test Plan & Acceptance Matrix
**Document ID:** `KW-DSH-004`  
**Deliverable:** D (Dashboard Experience Test Plan)  
**Parent Document:** [README.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/dashboard_experience/README.md)  
**Classification:** Quality Assurance & Automated Testing Standard  

---

## 1. Test Strategy & Acceptance Framework

Testing the Dashboard experience requires verifying truthful state visualization, action responsiveness, empty/error state handling, and keyboard accessibility.

Testing is structured across four tiers:
1. **Tier 1: Dashboard Store & Hook Unit Tests (Vitest):** Validates DTO parsing, state derivations, and action dispatch logic.
2. **Tier 2: Component Layout & State Tests (React Testing Library):** Validates rendering across all 5 operational states (Active, Unassigned, Disconnected, Degraded, Missing Asset).
3. **Tier 3: Keyboard Accessibility Scans (axe-core):** Scans for ARIA landmarks, visible focus rings, and screen-reader status announcements.
4. **Tier 4: Mocked IPC Integration Scenarios:** Exercises Play/Pause toggles, mute actions, and navigation triggers against mock Tauri invoke bridges.

> [!IMPORTANT]
> All test cases below represent approved verification specifications. Their status is marked as `NOT RUN (DESIGN PHASE)` pending runtime implementation in Phase 5.

---

## 2. Acceptance Criteria Verification Matrix (`DSH-001` through `DSH-015`)

| Test ID | Test Scenario | Target Subsystems | Observable Acceptance Criteria | Status |
| :--- | :--- | :--- | :--- | :--- |
| **`DSH-001`** | **Active Wallpaper Overview** | `DashboardView` Hero Panel| Active wallpaper title, author, resolution, and thumbnail render accurately from `DashboardBootstrapDTO`. | `NOT RUN (DESIGN PHASE)` |
| **`DSH-002`** | **Unassigned Onboarding State**| Empty State Card | When `activeWallpaper === null`, renders Welcome card with "Browse Wallpaper Library" CTA; 0 dummy metrics. | `NOT RUN (DESIGN PHASE)` |
| **`DSH-003`** | **Engine Disconnected Banner** | Network / IPC State | If Core is offline, renders "Connecting to Core Daemon..." warning banner; disables action buttons. | `NOT RUN (DESIGN PHASE)` |
| **`DSH-004`** | **Truthful FPS Presentation** | System Metrics Card | Real frame rate (e.g. 59.94 FPS) displayed when telemetry is active; omitted when telemetry is off; 0 fake values. | `NOT RUN (DESIGN PHASE)` |
| **`DSH-005`** | **Play/Pause Quick Action** | `PlaybackControlGroup` | Clicking "Pause" invokes `pause_all_playback`; button shows loading spinner until Core emits ACK; updates to "Resume". | `NOT RUN (DESIGN PHASE)` |
| **`DSH-006`** | **Mute/Unmute Quick Action** | `PlaybackControlGroup` | Clicking "Mute" invokes `set_global_mute`; updates volume icon to muted state; preserves volume slider level. | `NOT RUN (DESIGN PHASE)` |
| **`DSH-007`** | **Browse Library Shortcut** | Router Navigation | Clicking "Open Wallpaper Library" navigates router to `'library'`, focusing search bar immediately. | `NOT RUN (DESIGN PHASE)` |
| **`DSH-008`** | **Configure Displays Shortcut**| Router Navigation | Clicking "Configure Multi-Monitor" navigates router to `'displays'`, opening monitor topology canvas. | `NOT RUN (DESIGN PHASE)` |
| **`DSH-009`** | **Hardware Degraded Banner** | Degraded State Alert | When `engineState === 'degraded'`, renders alert card detailing graphics recovery; exposes "Re-anchor" button. | `NOT RUN (DESIGN PHASE)` |
| **`DSH-010`** | **Missing Asset Handling** | Corrupt File Card | If video file path is missing on disk, renders error card with "Select New Wallpaper"; zero unhandled exceptions. | `NOT RUN (DESIGN PHASE)` |
| **`DSH-011`** | **Wallpaper Playback Invariance**| Player Subprocess | Mounting Dashboard or switching views causes 0 frame drops or interruptions on active wallpapers. | `NOT RUN (DESIGN PHASE)` |
| **`DSH-012`** | **Keyboard Tab Reachability** | `DashboardView` (a11y) | Every action button and card trigger can be reached via `Tab`; visible 2px cyan focus ring (`#00F0FF`). | `NOT RUN (DESIGN PHASE)` |
| **`DSH-013`** | **Responsive 960px Layout** | Breakpoint Engine | In $960 \times 600\text{ px}$ compact window, hero panel and quick actions stack cleanly without horizontal overflow. | `NOT RUN (DESIGN PHASE)` |
| **`DSH-014`** | **Automated Accessibility Scan** | `axe-core` | Automated scan on `DashboardView` reports 0 critical or serious WCAG 2.1 AA violations. | `NOT RUN (DESIGN PHASE)` |
| **`DSH-015`** | **Zero Memory Leaks on Unmount** | Component Lifecycle | Navigating away from Dashboard cleans up `ipc://status-changed` listeners; leaves 0 dangling event handlers. | `NOT RUN (DESIGN PHASE)` |

---

## 3. Sample Vitest Dashboard Test Specification

```typescript
import { render, screen, fireEvent } from '@testing-library/react';
import { DashboardView } from './DashboardView';
import { useDashboardStore } from '../stores/dashboardStore';

describe('Dashboard Experience Specification (DSH-001, DSH-002, DSH-005)', () => {
  it('renders active wallpaper hero card when wallpaper is assigned', () => {
    useDashboardStore.setState({
      activeWallpaper: {
        id: 'wall-1',
        title: 'Cyberpunk Metropolis 4K',
        author: 'NeonForge',
        wallpaperType: 'video',
        thumbnailPath: 'mock-thumb.jpg',
        targetDisplayName: '\\\\.\\DISPLAY1',
        volume: 75,
        isMuted: false,
      },
    });

    render(<DashboardView />);
    expect(screen.getByText('Cyberpunk Metropolis 4K')).toBeInTheDocument();
    expect(screen.getByText(/NeonForge/)).toBeInTheDocument();
  });

  it('renders empty onboarding state when no wallpaper is configured', () => {
    useDashboardStore.setState({ activeWallpaper: null });
    render(<DashboardView />);
    expect(screen.getByText(/No active live wallpaper is currently assigned/i)).toBeInTheDocument();
    expect(screen.getByRole('button', { name: /browse wallpaper library/i })).toBeInTheDocument();
  });
});
```

