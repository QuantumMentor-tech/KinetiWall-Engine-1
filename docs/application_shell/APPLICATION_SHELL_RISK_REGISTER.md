# Application Shell Risk Register & Architectural Decisions
**Document ID:** `KW-SHL-005`  
**Deliverable:** E (Application Shell Risk Register)  
**Parent Document:** [README.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/application_shell/README.md)  
**Classification:** Core System Architecture / Operational Reliability Standard  

---

## 1. Application Shell Risk Register

Designing the top-level application shell and navigation framework introduces technical risks that must be addressed:

| Risk ID | Risk Summary & Failure Mode | Prob. | Impact | Severity | Architecture Mitigation & Prevention Strategy |
| :--- | :--- | :---: | :---: | :---: | :--- |
| **`RSK-SHL-01`** | **View Crash Brings Down Entire UI**<br>An unhandled rendering exception in an active view crashes the entire React application tree, closing window. | Med | High | **CRITICAL** | **In-Viewport Error Boundaries (`DEC-SHL-01`)**: `ContentViewport` is wrapped in a `ViewErrorBoundary`. A view failure displays a fallback card; navigation stays active. |
| **`RSK-SHL-02`** | **Navigation Resets Wallpaper Playback**<br>Switching views dispatches unnecessary IPC resets to Core, causing active live wallpapers to restart or flicker. | Med | High | **CRITICAL** | **Absolute Navigation Decoupling (`DEC-SHL-02`)**: View routing is strictly client-side presentation state; it dispatches zero lifecycle commands to Core players. |
| **`RSK-SHL-03`** | **Memory Retention across View Switches**<br>Navigating between views leaves zombie event listeners and heavy canvas buffers in memory, bloating UI RAM. | High | Med | **HIGH** | **Mandatory Cleanup on Unmount (`DEC-SHL-03`)**: All view hooks implement `useEffect` return cleanup functions, tearing down listeners and unmounting canvases. |
| **`RSK-SHL-04`** | **Visual Clipping at Window Min-Bounds**<br>User snaps UI to half-screen ($960\text{ px}$); navigation buttons or library cards clip horizontally offscreen. | Med | Med | **HIGH** | **Responsive Breakpoints & Auto-Fit Grids (`DEC-SHL-04`)**: Minimum width enforced at $960\text{ px}$; library grid dynamically scales from 2 to 6 columns. |
| **`RSK-SHL-05`** | **Flicker on Repeated Destination Clicks**<br>Clicking the active navigation button repeatedly triggers view re-mounting and re-fetching. | High | Low | **MEDIUM** | **Anti-Thrashing Navigation Guard (`DEC-SHL-05`)**: Navigation store checks `if (target === current) return;`, eliminating unnecessary re-renders. |
| **`RSK-SHL-06`** | **Context Loss on Navigation Switch**<br>User types a search query in Library, checks Settings, and returns; search query and scroll offset are lost. | High | Low | **MEDIUM** | **Zustand View Context Preservation (`DEC-SHL-06`)**: Search queries, filter selections, and scroll states are retained in persistent client stores. |
| **`RSK-SHL-07`** | **Windows 11 Snap Assist Blocked**<br>Custom borderless titlebar prevents native Windows 11 snap layout menus from appearing on Maximize button hover. | Med | Med | **MEDIUM** | **Native Titlebar Snap Integration (`DEC-SHL-07`)**: Custom header buttons implement native `WM_NCHITTEST` handling (`HTMAXBUTTON`) via Tauri decorum. |
| **`RSK-SHL-08`** | **Deceptive Unimplemented Navigation**<br>Navigation sidebar displays "Workshop" button; user clicks it and is redirected to an empty or broken screen. | Med | Med | **MEDIUM** | **Honest Disabled Availability States (`DEC-SHL-08`)**: Unavailable features are rendered with explicit disabled styling and informative tooltips; navigation is blocked. |

---

## 2. Architecture Decision Records (Application Shell: `DEC-SHL-01` to `08`)

The following formal decisions govern all application shell and navigation implementations across KinetiWall:

### `DEC-SHL-01`: Top-Level View Error Boundaries
* **Context:** Third-party wallpaper packages can contain corrupted metadata or invalid image files that trigger runtime JavaScript exceptions.
* **Decision:** Wrap `ContentViewport` in a dedicated `ViewErrorBoundary` component.
* **Consequences:** View crashes are quarantined within the content area; the user can freely navigate to other sections or reload the view without restarting the application.

### `DEC-SHL-02`: Zero IPC Emission from Navigation Changes
* **Context:** Navigating between views is a presentation concern and must not disturb native engine playback.
* **Decision:** Route transitions in `routerStore` execute entirely in client memory; they must never invoke IPC lifecycle commands on `kinetiwall-core.exe`.
* **Consequences:** Guarantees 100% wallpaper playback continuity across all displays during settings navigation.

### `DEC-SHL-03`: Strict Event Subscription Cleanup on Unmount
* **Context:** Mounting views that subscribe to Tauri IPC events can cause memory leaks and duplicate handler execution if not released.
* **Decision:** All view-level subscriptions must be encapsulated in React custom hooks with explicit cleanup teardowns in `useEffect`.
* **Consequences:** Eliminates memory leaks and prevents phantom state updates after route switching.

### `DEC-SHL-04`: Enforce $960 \times 600\text{ px}$ Minimum Window Usability
* **Context:** Windows desktop users frequently snap windows to 50% screen splits.
* **Decision:** Enforce `minWidth: 960` and `minHeight: 600` in Tauri configuration; layout containers use CSS flexbox and `minmax()` grid definitions.
* **Consequences:** Guarantees zero horizontal clipping or hidden action controls across all supported desktop window configurations.

### `DEC-SHL-05`: Anti-Thrashing Guard on Navigation Actions
* **Context:** Rapidly double-clicking or re-clicking active navigation items causes unnecessary re-mounting and UI flicker.
* **Decision:** The router store ignores navigation requests targeting the already active route key.
* **Consequences:** Guarantees rock-solid visual stability and zero redundant DOM re-renders.

### `DEC-SHL-06`: In-Memory View State Retention in Zustand
* **Context:** Users expect search filters, scroll offsets, and active tabs to remain intact when briefly switching views.
* **Decision:** Store view context in dedicated Zustand stores (`libraryStore`, `settingsStore`) rather than local component state.
* **Consequences:** Delivers a fluid, desktop-native user experience with instant context restoration.

### `DEC-SHL-07`: Native Window Header Bar with Snap Assist
* **Context:** Borderless windows often break native Windows 11 titlebar gestures (dragging, snapping, double-click to maximize).
* **Decision:** Implement a custom header bar utilizing `data-tauri-drag-region` and native maximize button hit-testing.
* **Consequences:** Provides custom Neon Glass cyberpunk styling while preserving 100% native Windows window-management usability.

### `DEC-SHL-08`: Honest Disabled Availability for V1.0 Non-Goals
* **Context:** Online wallpaper workshops and cloud syncing are deferred to future releases (`KW-NG-002`).
* **Decision:** Declare deferred routes with `isAvailable: false`. Render with disabled visual styling (`opacity-40 cursor-not-allowed`) and informative offline tooltips.
* **Consequences:** Sets transparent, honest expectations for users without confusing dead-end navigation links.

---

## 3. Open Engineering Review Items

| Item ID | Description | Current Architectural Status | Target Phase |
| :--- | :--- | :--- | :---: |
| **`REV-SHL-01`** | **Collapsible Navigation Rail on Compact Viewports:** Evaluate whether the 200px sidebar should collapse to icon-only (64px) at widths $< 1000\text{ px}$. | Standard 200px width tested cleanly; icon-only rail planned as progressive enhancement. | Phase 5 (UI Implementation) |
| **`REV-SHL-02`** | **Tauri Window Shadow & Borderless Resizing:** Verify custom borderless window resizing ergonomics on Windows 10 vs Windows 11. | Native Tauri borderless window shadows verified; resizing edge bounds to be calibrated. | Phase 5 (UI Implementation) |
| **`REV-SHL-03`** | **Breadcrumb Navigation for Nested Settings:** Determine whether deep property customization requires a breadcrumb bar in the view header. | Flat view model approved for V1.0; modal dialogs handle deep customization without nesting. | Phase 5 (UI Implementation) |

