# UI Component Test Strategy & Acceptance Matrix
**Document ID:** `KW-UI-004`  
**Deliverable:** D (Component Testing Strategy)  
**Parent Document:** [README.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/ui/README.md)  
**Classification:** Quality Assurance & Automated Testing Standard  

---

## 1. Test Strategy & Acceptance Framework

Testing user interface components in KinetiWall requires validating visual rendering, keyboard accessibility, state updates, and IPC service boundaries without requiring live Windows GPU players.

Testing is divided into four rigorous tiers:
1. **Tier 1: Component Unit Tests (Vitest + React Testing Library):** Validates prop variations, visual states, event callbacks, and DOM element attributes.
2. **Tier 2: Keyboard & Accessibility Audits (axe-core):** Automated scans for WCAG 2.1 AA violations, missing ARIA labels, and contrast deficiencies.
3. **Tier 3: Visual Regression Testing:** Baseline snapshot comparisons verifying padding, borders, shadows, and theme consistency.
4. **Tier 4: Mocked IPC Integration Tests:** Verifies that composed components submit correct payloads to `@tauri-apps/api` bridges and render loading/error states properly.

> [!IMPORTANT]
> All test cases below represent approved verification specifications. Their status is marked as `NOT RUN (DESIGN PHASE)` pending runtime implementation in Phase 4.

---

## 2. Acceptance Criteria Verification Matrix (`UI-001` through `UI-015`)

| Test ID | Test Scenario | Target Component | Observable Acceptance Criteria | Status |
| :--- | :--- | :--- | :--- | :--- |
| **`UI-001`** | **Button Variants & Focus Ring** | `Button` | All 5 variants render correct design token colors; tabbing onto button displays visible 2px cyan focus ring. | `NOT RUN (DESIGN PHASE)` |
| **`UI-002`** | **Button Loading State** | `Button` (`isLoading`) | Setting `isLoading={true}` renders animated spinner, disables button, and suppresses click events. | `NOT RUN (DESIGN PHASE)` |
| **`UI-003`** | **TextInput Error State** | `TextInput` | Passing `error="Invalid path"` renders 1px magenta border (`#FF0055`) and accessible error message below input. | `NOT RUN (DESIGN PHASE)` |
| **`UI-004`** | **Switch Keyboard Toggle** | `Switch` | Pressing `Space` or `Enter` on focused switch toggles `checked` state and dispatches `onChange` callback. | `NOT RUN (DESIGN PHASE)` |
| **`UI-005`** | **Slider Arrow Adjustment** | `Slider` | Pressing `ArrowRight` increments value by `step`; updates `aria-valuenow` and calls `onChange`. | `NOT RUN (DESIGN PHASE)` |
| **`UI-006`** | **Modal Focus Trap** | `ConfirmationDialog` | Tabbing wraps within modal buttons; background elements cannot be reached; `Escape` closes dialog. | `NOT RUN (DESIGN PHASE)` |
| **`UI-007`** | **Focus Restoration on Modal Close**| `ConfirmationDialog` | Closing modal restores active keyboard focus back to the button that originally triggered it. | `NOT RUN (DESIGN PHASE)` |
| **`UI-008`** | **WallpaperCard Safe Escaping** | `WallpaperCard` | Malicious payload `<script>alert(1)</script>` in title is rendered safely as plain text; zero XSS execution. | `NOT RUN (DESIGN PHASE)` |
| **`UI-009`** | **WallpaperCard Apply Action** | `WallpaperCard` | Clicking "Apply" invokes `onApply(displayId)`; card enters loading state until IPC completes. | `NOT RUN (DESIGN PHASE)` |
| **`UI-010`** | **MonitorSelector Layout** | `MonitorSelector` | Dual-monitor configuration renders displays with accurate relative aspect ratios and primary badges. | `NOT RUN (DESIGN PHASE)` |
| **`UI-011`** | **PlaybackControlGroup Mute** | `PlaybackControlGroup`| Clicking mute button toggles mute state, reflects muted icon, and disables volume slider visually. | `NOT RUN (DESIGN PHASE)` |
| **`UI-012`** | **SearchBar Debouncing** | `SearchBar` | Rapid typing invokes `onChange` exactly once after the configured debounce interval (e.g. 300 ms). | `NOT RUN (DESIGN PHASE)` |
| **`UI-013`** | **Automated a11y Audit** | All Shared Components | Automated `axe-core` scan reports 0 critical or serious accessibility violations across all states. | `NOT RUN (DESIGN PHASE)` |
| **`UI-014`** | **Reduced-Motion Enforcement** | Tailwind Animations | Under `prefers-reduced-motion: reduce`, all transitions snap instantly ($\le 0.01\text{ ms}$). | `NOT RUN (DESIGN PHASE)` |
| **`UI-015`** | **Zero Memory Leaks on Unmount** | All Shared Components | Rapid mounting and unmounting of components in testing harness leaves 0 dangling event listeners. | `NOT RUN (DESIGN PHASE)` |

---

## 3. Sample Vitest Component Test Specification

```typescript
import { render, screen, fireEvent } from '@testing-library/react';
import { Button } from './Button';

describe('Button Component (UI-001, UI-002)', () => {
  it('renders primary variant with accessible focus ring', () => {
    render(<Button variant="primary">Apply Wallpaper</Button>);
    const button = screen.getByRole('button', { name: /apply wallpaper/i });
    expect(button).toBeInTheDocument();
    expect(button).toHaveClass('bg-neon-cyan');
  });

  it('disables click and shows spinner when loading', () => {
    const handleClick = vi.fn();
    render(<Button isLoading onClick={handleClick}>Save</Button>);
    const button = screen.getByRole('button', { name: /save/i });
    expect(button).toBeDisabled();
    fireEvent.click(button);
    expect(handleClick).not.toHaveBeenCalled();
    expect(screen.getByTestId('spinner')).toBeInTheDocument();
  });
});
```

