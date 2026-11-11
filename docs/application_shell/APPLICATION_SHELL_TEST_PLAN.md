# Application Shell Test Plan & Acceptance Matrix
**Document ID:** `KW-SHL-004`  
**Deliverable:** D (Application Shell Test Plan)  
**Parent Document:** [README.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/application_shell/README.md)  
**Classification:** Quality Assurance & Automated Testing Standard  

---

## 1. Test Strategy & Acceptance Framework

Testing the root application shell and navigation architecture requires validating visual layout stability, keyboard accessibility, route state machines, and window resize adaptability.

Testing is divided into four rigorous tiers:
1. **Tier 1: Router Store Unit Tests (Vitest):** Validates route transitions, active-key derivations, anti-thrashing no-ops, and context preservation.
2. **Tier 2: Component Layout Tests (React Testing Library):** Validates `AppShell`, `NavigationSidebar`, and `WindowHeaderBar` rendering under different prop conditions.
3. **Tier 3: Keyboard Accessibility Scans (axe-core):** Automated checks for ARIA landmarks, `aria-current`, visible focus rings, and tab sequence.
4. **Tier 4: Responsive Viewport Tests (Playwright Component Testing):** Verifies layout scaling across all 4 breakpoint tiers ($960\text{ px}$ to $3840\text{ px}$).

> [!IMPORTANT]
> All test cases below represent approved verification specifications. Their status is marked as `NOT RUN (DESIGN PHASE)` pending runtime implementation in Phase 5.

---

## 2. Acceptance Criteria Verification Matrix (`SHL-001` through `SHL-015`)

| Test ID | Test Scenario | Target Subsystems | Observable Acceptance Criteria | Status |
| :--- | :--- | :--- | :--- | :--- |
| **`SHL-001`** | **AppShell Root Rendering** | `AppShell` Container | Shell mounts within $\le 350\text{ ms}$; renders custom header, navigation sidebar, and library content viewport. | `NOT RUN (DESIGN PHASE)` |
| **`SHL-002`** | **Active Navigation Indicator** | `NavigationSidebar` | Selecting "Displays" shifts active cyan border strip and `aria-current="page"` to Displays route; Library unselected. | `NOT RUN (DESIGN PHASE)` |
| **`SHL-003`** | **Anti-Thrashing Navigation** | `routerStore` | Clicking the currently active route key (e.g. "Library") is a strict no-op; 0 re-mounts, 0 refetches. | `NOT RUN (DESIGN PHASE)` |
| **`SHL-004`** | **Honest Unavailable Route State**| Disabled Navigation | Clicking "Online Hub" is blocked; route does not change; tooltip displays "Online features are disabled in V1.0". | `NOT RUN (DESIGN PHASE)` |
| **`SHL-005`** | **Search Filter Context Retention**| View State Preservation| Entering search string in Library, navigating to Settings, and returning to Library retains exact search string. | `NOT RUN (DESIGN PHASE)` |
| **`SHL-006`** | **Keyboard Sidebar Navigation** | `NavigationSidebar` (a11y)| Users can navigate sidebar using `ArrowDown`/`ArrowUp`; `Enter` activates destination; visible 2px cyan focus ring. | `NOT RUN (DESIGN PHASE)` |
| **`SHL-007`** | **Window Min-Bounds Usability** | Responsive Shell ($960\text{px}$)| At $960 \times 600\text{ px}$ viewport, all navigation buttons and library cards render without horizontal scrollbars or clipping. | `NOT RUN (DESIGN PHASE)` |
| **`SHL-008`** | **Wide Viewport Adaptation** | Responsive Shell ($1600\text{px}$)| At $1600\text{ px}$ width, library grid expands smoothly from 2 to 4 columns; monitor selector canvas expands. | `NOT RUN (DESIGN PHASE)` |
| **`SHL-009`** | **View Error Boundary Containment**| `ViewErrorBoundary` | Injecting JavaScript exception into Library view renders in-viewport error card; sidebar remains fully interactive. | `NOT RUN (DESIGN PHASE)` |
| **`SHL-010`** | **Zero Duplicate Core Spawns** | Process Architecture | Launching, minimizing, and restoring `kinetiwall-ui.exe` spawns 0 duplicate `kinetiwall-core.exe` background processes. | `NOT RUN (DESIGN PHASE)` |
| **`SHL-011`** | **Wallpaper Continuity on Nav** | Player Subprocess | Navigating between Library, Displays, and Settings produces zero frame drops or restarts on active video wallpapers. | `NOT RUN (DESIGN PHASE)` |
| **`SHL-012`** | **Automated Accessibility Audit** | `axe-core` Scan | Automated scan on `AppShell` reports 0 critical or serious WCAG 2.1 AA violations. | `NOT RUN (DESIGN PHASE)` |
| **`SHL-013`** | **Window Drag Region** | `WindowHeaderBar` | Titlebar region declares `data-tauri-drag-region`; dragging moves desktop window smoothly. | `NOT RUN (DESIGN PHASE)` |
| **`SHL-014`** | **Reduced-Motion Enforcement** | Tailwind Animations | Under `prefers-reduced-motion: reduce`, view transition fade durations snap to $0\text{ ms}$. | `NOT RUN (DESIGN PHASE)` |
| **`SHL-015`** | **Zero Memory Leaks on Unmount** | Navigation Lifecycle | Navigating between routes 100 times leaves 0 dangling IPC event listeners or uncollected DOM nodes. | `NOT RUN (DESIGN PHASE)` |

---

## 3. Sample Vitest Navigation Store Test Specification

```typescript
import { renderHook, act } from '@testing-library/react';
import { useRouterStore } from './routerStore';

describe('Router Store Specification (SHL-002, SHL-003, SHL-004)', () => {
  beforeEach(() => {
    useRouterStore.getState().resetRoute();
  });

  it('initializes with default library route', () => {
    const { result } = renderHook(() => useRouterStore());
    expect(result.current.activeRoute).toBe('library');
  });

  it('updates active route when navigating to valid destination', () => {
    const { result } = renderHook(() => useRouterStore());
    act(() => {
      result.current.navigate('displays');
    });
    expect(result.current.activeRoute).toBe('displays');
  });

  it('rejects navigation to disabled/unavailable route', () => {
    const { result } = renderHook(() => useRouterStore());
    act(() => {
      result.current.navigate('workshop');
    });
    // Must remain on library; cannot navigate to unavailable feature
    expect(result.current.activeRoute).toBe('library');
  });
});
```

