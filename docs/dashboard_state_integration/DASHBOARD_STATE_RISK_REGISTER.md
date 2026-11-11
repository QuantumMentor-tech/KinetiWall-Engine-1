# Dashboard State Integration Risk Register & ADRs
**Document ID:** `KW-DSI-005`  
**Phase:** 4 (UI/UX Design System & Dashboard)  
**Step:** 24 of 90  
**Status:** Approved Architectural Specification  
**Classification:** Risk Governance & Architectural Decision Records  

---

## 1. Executive Summary & Governance Scope

This register documents the operational risks, failure modes, mitigations, and formal Architecture Decision Records (ADRs) governing state synchronization, IPC reliability, and interaction safety between the KinetiWall Dashboard (`src-ui/views/DashboardView.tsx`) and the native Core engine (`kinetiwall-core.exe`).

---

## 2. Risk Analysis Matrix (`RSK-DSI-01` through `RSK-DSI-08`)

| Risk ID | Risk Description | Severity | Likelihood | Impact Area | Mitigation & Architectural Defense |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **`RSK-DSI-01`** | **Stale Out-of-Order IPC Responses:** Network or thread scheduling delays cause an earlier command response to arrive after a newer one, regressing the UI state. | High | Medium | UI Consistency | Enforce monotonic 64-bit sequence counters (`requestId: u64`); discard responses where `requestId < highestCommitted` (`DEC-DSI-01`). |
| **`RSK-DSI-02`** | **Rapid Click Ingress Flooding:** Fast repeated clicks or key repeats flood the IPC channel with redundant state mutations. | High | High | IPC Pipe / Core Stability | Synchronously set `disabled={true}` on controls upon click; enforce a $200\text{ ms}$ debounce cooldown per action (`DEC-UI-04`). |
| **`RSK-DSI-03`** | **IPC Channel Disconnection Crash:** Unexpected termination of `kinetiwall-core.exe` causes unhandled promise rejections in the UI. | Critical | Low | App Stability | Wrap IPC calls in transport-level try/catch blocks; trigger circuit breaker with exponential backoff (`DEC-DSI-04`). |
| **`RSK-DSI-04`** | **Event Listener Closure Leaks:** Repeated window open/close or component remount cycles accumulate Tauri event listeners in memory. | Medium | Medium | Frontend Memory | Deterministically execute `unlisten()` callbacks in React `useEffect` cleanup handlers (`DEC-DSI-05`). |
| **`RSK-DSI-05`** | **Display Topology Desynchronization:** Monitor index changes (e.g. plugging in a third screen) cause wallpaper assignments to shift incorrectly. | High | Medium | Multi-Monitor Setup | Index display slots by permanent Windows Device IDs (`displayId`), never transient monitor indices (`DEC-DSI-06`). |
| **`RSK-DSI-06`** | **Premature Timeout on Heavy Profiles:** Complex wallpaper profiles with large 4K video assets exceed default $5000\text{ ms}$ timeout, causing false rollback. | Medium | Medium | User Experience | Core returns immediate `Status::InProgress` with progress events; UI extends timeout dynamically for profile loads (`DEC-DSI-02`). |
| **`RSK-DSI-07`** | **Telemetry Stream Saturation:** Unthrottled 60 FPS telemetry broadcasts saturate IPC and cause UI frame drops. | Medium | Low | Frame Presentation | Core throttles telemetry heartbeats to $1\text{ Hz}$, active only when dashboard window is focused (`DEC-DSI-07`). |
| **`RSK-DSI-08`** | **Sensitive Path Disclosure in Logs:** Unsanitized file system paths in error toasts or crash logs leak usernames or private directories. | Medium | High | User Privacy | Pass all error strings and file URIs through a regex privacy redaction filter prior to rendering or logging (`DEC-DSI-08`). |

---

## 3. Architecture Decision Records (`DEC-DSI-01` through `DEC-DSI-08`)

### `DEC-DSI-01`: Monotonic 64-Bit Request Sequence Counter
* **Context:** In an asynchronous multi-process architecture, IPC message ordering across separate OS threads is non-deterministic under heavy system load.
* **Decision:** All state mutation commands dispatched from `kinetiwall-ui.exe` carry a monotonic unsigned 64-bit integer (`requestId`). The Zustand store records `highestCommittedSequence` and drops any response or broadcast event that reflects an earlier sequence number.
* **Consequences:** Eliminates stale response regressions without requiring expensive blocking roundtrips.

### `DEC-DSI-02`: Optimistic UI Updates with Deterministic Rollback Snapshotting
* **Context:** Desktop users expect instantaneous visual feedback when toggling primary controls (Play/Pause, Mute). Waiting for IPC roundtrips creates perceived lag.
* **Decision:** The UI applies target states optimistically while locking ingress. Before mutating the store, a state snapshot is recorded in `pendingActions`. If the Core rejects the action or times out ($5000\text{ ms}$), the store automatically restores the snapshot and displays an accessible toast.
* **Consequences:** Near-zero perceived latency for successful actions; robust error recovery for failures.

### `DEC-DSI-03`: Transport-Gate Zod Schema Validation
* **Context:** Changes in Core DTO formats or transmission errors could introduce undefined fields, causing unhandled runtime crashes in React.
* **Decision:** Validate all IPC responses and event payloads using `Zod` schemas at the Tauri invocation wrapper before passing data into Zustand reducers.
* **Consequences:** Traps payload mismatches at the boundary; provides actionable diagnostic logs without crashing the view.

### `DEC-DSI-04`: Bounded Circuit Breaker with Exponential Backoff
* **Context:** If `kinetiwall-core.exe` crashes or is killed by the user, the UI must not hammer the non-existent pipe with continuous retries.
* **Decision:** Implement a three-state circuit breaker (`Closed`, `Open`, `Half-Open`). After 3 consecutive IPC failures, trip the breaker to `Open`, pause for $500\text{ ms}$, and retry with $2\times$ backoff up to $2000\text{ ms}$. If still unresolvable, enter permanent disconnected mode with a user restart button.
* **Consequences:** Prevents CPU spinning and IPC resource starvation during engine failure.

### `DEC-DSI-05`: Deterministic Lifecycle Binding for IPC Event Listeners
* **Context:** Tauri event listeners (`listen()`) persist until explicitly unsubscribed. In an ephemeral UI, repeatedly mounting views could leak listener closures.
* **Decision:** Encapsulate all event subscriptions in a lifecycle manager that registers listeners upon store initialization and unlistens on window `beforeunload` or view unmount.
* **Consequences:** Zero memory growth across repeated view transitions.

### `DEC-DSI-06`: Hardware Display ID Mapping over Logical Indices
* **Context:** Windows logical monitor indices (`\\.\DISPLAY1`, `\\.\DISPLAY2`) change dynamically upon display hotplugging or sleep/wake events.
* **Decision:** Bind display slots to persistent hardware identifiers (`DISPLAY_DEVICE_ID` derived from EDID and GPU bus path).
* **Consequences:** Wallpaper assignments remain pinned to the correct physical monitor regardless of Windows display re-indexing.

### `DEC-DSI-07`: 1 Hz Throttled Passive Telemetry Stream
* **Context:** High-frequency telemetry (framerate, memory) creates unnecessary IPC chatter when the user is not actively viewing the dashboard.
* **Decision:** Core streams telemetry at a maximum frequency of $1\text{ Hz}$, and suspends broadcasts entirely when the dashboard window is minimized or blurred.
* **Consequences:** Conforms strictly to $0\%\text{ idle CPU}$ utilization targets (`KW-NF-001`).

### `DEC-DSI-08`: Path-Redacting Privacy Filter on Diagnostic Logging
* **Context:** Diagnostic logs and error toasts may contain absolute file paths (e.g. `C:\Users\JohnDoe\AppData\Local\...`).
* **Decision:** All error strings and asset paths pass through a privacy sanitizer that replaces personal user directories with `[USER_DIR]` before rendering or emitting telemetry.
* **Consequences:** Enforces strict privacy compliance (`KW-OBS-001`).

---

## 4. Open Engineering Review Items

1. **Dynamic Timeout Scaling for 8K Video Profiles:** Evaluate whether profile switches involving 8K video decoders require an extended timeout threshold ($8000\text{ ms}$) instead of the standard $5000\text{ ms}$.
2. **Audio Peak Metering Stream:** Assess whether live audio visualization bars on the Hero panel require a dedicated low-overhead shared memory ring buffer instead of Tauri IPC events if added in future releases.

