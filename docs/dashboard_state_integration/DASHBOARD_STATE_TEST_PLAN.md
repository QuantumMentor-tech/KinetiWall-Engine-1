# Dashboard State Integration Test Plan
**Document ID:** `KW-DSI-004`  
**Phase:** 4 (UI/UX Design System & Dashboard)  
**Step:** 24 of 90  
**Status:** Approved Architectural Specification  
**Classification:** Quality Assurance & Automated Verification Strategy  

---

## 1. Executive Summary & Strategy

This test plan defines the automated verification strategy for KinetiWall's Dashboard State Integration, Live Status Synchronization, and Reliability architecture.

The test strategy employs a four-tier testing hierarchy to validate that all state mutations, asynchronous IPC interactions, error rollbacks, and memory cleanup guarantees function deterministically across all supported Windows environments.

```
+-------------------------------------------------------------------------+
| Tier 4: System & E2E Tests (Playwright / Tauri Driver)                  |
| Cold-start bootstrap latency, window lifecycle deallocation, multi-     |
| monitor hotplug simulation.                                             |
+-------------------------------------------------------------------------+
                                    ^
+-------------------------------------------------------------------------+
| Tier 3: Integration Tests (Vitest + Tauri IPC Mock Harness)             |
| Passive event stream sync, stale response dropping, out-of-order seq.   |
+-------------------------------------------------------------------------+
                                    ^
+-------------------------------------------------------------------------+
| Tier 2: Component Tests (React Testing Library + @testing-library/user) |
| Action lifecycle state machine, ingress disabling, optimistic UI, rollbacks.|
+-------------------------------------------------------------------------+
                                    ^
+-------------------------------------------------------------------------+
| Tier 1: Unit Tests (Vitest + Zod Schema Validator)                      |
| Zustand store reducers, monotonic sequence counter, payload validation. |
+-------------------------------------------------------------------------+
```

---

## 2. Acceptance Criteria Matrix (`DSI-001` through `DSI-015`)

All tests are currently marked **`NOT RUN (DESIGN PHASE)`** pending Phase 4 implementation:

| Criteria ID | Requirement Ref | Verification Scope | Expected Behavior / Verification Standard | Status |
| :--- | :--- | :--- | :--- | :--- |
| **`DSI-001`** | `KW-NF-004` | Cold Bootstrap Latency | Single atomic query `REQ_GET_DASHBOARD_STATE` resolves and populates store in $\le 50\text{ ms}$. | `NOT RUN (DESIGN PHASE)` |
| **`DSI-002`** | `KW-NF-001` | Zero Polling Invariance | Zero `setInterval`, `setTimeout`, or `requestAnimationFrame` polling loops active across IPC boundary. | `NOT RUN (DESIGN PHASE)` |
| **`DSI-003`** | `KW-NG-001` | Single Source of Truth | UI never mutates persistent state independently; all changes originate from Core IPC events. | `NOT RUN (DESIGN PHASE)` |
| **`DSI-004`** | `DEC-UI-04` | Action Ingress Disabling | Triggering Play/Pause or Mute synchronously sets `disabled={true}` and blocks duplicate clicks. | `NOT RUN (DESIGN PHASE)` |
| **`DSI-005`** | `DEC-DSI-01` | Stale Response Dropping | Out-of-order IPC responses with `requestId < highestCommitted` are discarded without mutating state. | `NOT RUN (DESIGN PHASE)` |
| **`DSI-006`** | `DEC-DSI-02` | Deterministic Rollback | IPC command timeout ($5000\text{ ms}$) or error reverts optimistic UI change to snapshot state. | `NOT RUN (DESIGN PHASE)` |
| **`DSI-007`** | `KW-OBS-002` | Health vs Playback Decoupling | Degraded engine health does not falsely mark wallpaper playback as `Stopped` or `Error`. | `NOT RUN (DESIGN PHASE)` |
| **`DSI-008`** | `KW-F-004` | Occlusion State Mapping | Fullscreen app occlusion correctly updates dashboard badge to `Occluded` without triggering errors. | `NOT RUN (DESIGN PHASE)` |
| **`DSI-009`** | `KW-F-002` | Multi-Monitor Slot Isolation | Crash or disconnect on Display 2 marks slot as `Detached` or `Faulted` while Display 1 remains `Active`. | `NOT RUN (DESIGN PHASE)` |
| **`DSI-010`** | `DEC-DSI-03` | Schema Validation Barrier | Corrupt/malformed IPC payloads are rejected by Zod schema without crashing React component tree. | `NOT RUN (DESIGN PHASE)` |
| **`DSI-011`** | `DEC-DSI-04` | Bounded Circuit Breaker | 3 consecutive IPC timeouts trigger circuit breaker with exponential backoff ($500\text{ ms}, 1000\text{ ms}, 2000\text{ ms}$). | `NOT RUN (DESIGN PHASE)` |
| **`DSI-012`** | `KW-NG-002` | Ephemeral UI Deallocation | Closing UI window deallocates 100% of webview memory ($0.0\text{ MB}$ RAM); player continues rendering. | `NOT RUN (DESIGN PHASE)` |
| **`DSI-013`** | `KW-UI-003` | Keyboard Ingress Lockout | Pressing `Space` repeatedly during pending playback toggle emits only 1 IPC command until settled. | `NOT RUN (DESIGN PHASE)` |
| **`DSI-014`** | `DEC-DSI-05` | Event Listener Cleanup | Unmounting `DashboardView` deterministically invokes all Tauri `unlisten()` functions. | `NOT RUN (DESIGN PHASE)` |
| **`DSI-015`** | `KW-OBS-001` | Telemetry Privacy Redaction | Error logs emitted from dashboard state failures contain redacted user paths (`[REDACTED]`). | `NOT RUN (DESIGN PHASE)` |

---

## 3. Automated Test Suite Specifications

### 3.1 Unit Test: Monotonic Request Sequence Counter (`Vitest`)

```typescript
import { describe, it, expect, beforeEach } from 'vitest';
import { useDashboardStore } from '../../src-ui/stores/dashboardStore';

describe('Dashboard Store: Monotonic Sequence Guardrails', () => {
  beforeEach(() => {
    useDashboardStore.getState().resetStore();
  });

  it('DSI-005: should drop stale out-of-order action responses', () => {
    const store = useDashboardStore.getState();

    // 1. Dispatch request 1 (seq = 101)
    const req1 = store.generateNextRequestId();
    expect(req1).toBe(101n);
    store.setPlaybackOptimistic('Paused', req1);

    // 2. Dispatch request 2 (seq = 102)
    const req2 = store.generateNextRequestId();
    expect(req2).toBe(102n);
    store.setPlaybackOptimistic('Playing', req2);

    // 3. Receive out-of-order confirmation for req2 FIRST
    store.commitActionSuccess(req2, { globalState: 'Playing' });
    expect(store.playback.globalState).toBe('Playing');
    expect(store.highestCommittedSequence).toBe(102n);

    // 4. Stale confirmation for req1 arrives LATER
    store.commitActionSuccess(req1, { globalState: 'Paused' });

    // 5. Must NOT revert to Paused!
    expect(store.playback.globalState).toBe('Playing');
    expect(store.highestCommittedSequence).toBe(102n);
  });
});
```

### 3.2 Component Test: Ingress Disabling & Rollback (`React Testing Library`)

```typescript
import { render, screen, fireEvent, waitFor } from '@testing-library/react';
import { vi, describe, it, expect } from 'vitest';
import { HeroActivePanel } from '../../src-ui/components/dashboard/HeroActivePanel';
import * as tauriIpc from '@tauri-apps/api/core';

vi.mock('@tauri-apps/api/core');

describe('HeroActivePanel Action Lifecycle', () => {
  it('DSI-004 & DSI-006: disables ingress on click and rolls back upon IPC failure', async () => {
    // Mock IPC rejection
    vi.spyOn(tauriIpc, 'invoke').mockRejectedValueOnce(new Error('IPC_TIMEOUT_EXCEEDED'));

    render(<HeroActivePanel />);
    const playPauseBtn = screen.getByRole('button', { name: /pause wallpaper/i });

    // Initial state: Idle and enabled
    expect(playPauseBtn).not.toBeDisabled();

    // Trigger action
    fireEvent.click(playPauseBtn);

    // Ingress must immediately lock
    expect(playPauseBtn).toBeDisabled();
    expect(playPauseBtn).toHaveAttribute('aria-busy', 'true');

    // Wait for failure rollback
    await waitFor(() => {
      expect(screen.getByText(/unable to update playback/i)).toBeInTheDocument();
    });

    // Control returns to idle and re-enables ingress
    expect(playPauseBtn).not.toBeDisabled();
    expect(playPauseBtn).toHaveAttribute('aria-busy', 'false');
  });
});
```

### 3.3 Integration Test: IPC Event Stream Synchronization

```typescript
import { describe, it, expect } from 'vitest';
import { mockIPC } from '@tauri-apps/api/mocks';
import { initializeDashboardSync } from '../../src-ui/stores/dashboardStore';

describe('Dashboard Integration: IPC Event Stream', () => {
  it('DSI-008: maps occlusion event from core to dashboard badge', async () => {
    let statusCallback: (evt: any) => void = () => {};
    mockIPC((cmd, args) => {
      if (cmd === 'listen' && args.event === 'ipc://status-changed') {
        statusCallback = args.handler;
      }
    });

    const store = useDashboardStore.getState();
    await initializeDashboardSync(store);

    // Simulate Core pushing occlusion event
    statusCallback({
      payload: {
        globalState: 'Occluded',
        timestamp: Date.now(),
        reason: 'FULLSCREEN_FOREGROUND_DETECTED',
      },
    });

    expect(store.playback.globalState).toBe('Occluded');
  });
});
```

