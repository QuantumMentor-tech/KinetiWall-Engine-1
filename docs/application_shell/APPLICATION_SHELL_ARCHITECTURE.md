# Application Shell Architecture Specification
**Document ID:** `KW-SHL-001`  
**Deliverable:** A (Application Shell Architecture)  
**Parent Document:** [README.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/application_shell/README.md)  
**Classification:** Frontend Architecture / Application Shell Standard  

---

## 1. Structural Layout Blueprint

The root application shell for KinetiWall (`kinetiwall-ui.exe`) establishes a rigid three-panel structure designed specifically for desktop live wallpaper administration:

```
+-------------------------------------------------------------------------------------------------------+
| TOP BAR: Custom Neon Glass Header Bar (Height: 48px)                                                  |
| [KinetiWall Neon Logo]  KinetiWall Engine      [Status: Active (60 FPS)]    [Settings] [ - ] [ □ ] [ ✕ ]|
+-------------------------------------------------------------------------------------------------------+
| SIDEBAR: Navigation Rail (Width: 240px) | MAIN CONTENT VIEWPORT (Fluid Flexbox / Scrollable)           |
|                                         |                                                             |
|  [Library]        Wallpaper Gallery     |  +-------------------------------------------------------+  |
|  [Displays]       Monitor Topology      |  | View Header: Active Section Title + Action Controls   |  |
|  [Performance]    Hardware Metrics      |  +-------------------------------------------------------+  |
|  [Settings]       Engine Configuration  |  |                                                       |  |
|  -------------------------------------  |  | Active View Content Container                         |  |
|  [Workshop]       (Disabled/Offline)    |  | (Library Grid / Monitor Canvas / Settings Forms)      |  |
|                                         |  |                                                       |  |
|  BOTTOM ANCHOR:                         |  |                                                       |  |
|  [Engine State: Connected] [Version 1.0]|  +-------------------------------------------------------+  |
+-------------------------------------------------------------------------------------------------------+
```

### 1.1 Shell Component Hierarchy

* **`AppShell` (`src-ui/components/layout/AppShell.tsx`):**
  - Top-level layout container binding the header, sidebar navigation, and main content outlet.
  - Enforces overall window min-bounds ($960 \times 600\text{ px}$) and background styling (`#0A0B10`).
* **`WindowHeaderBar` (`src-ui/components/layout/WindowHeaderBar.tsx`):**
  - Integrated custom window titlebar with drag region (`data-tauri-drag-region`).
  - Displays the KinetiWall brand mark, engine runtime status badge, and window control buttons (Minimize, Maximize, Close).
* **`NavigationSidebar` (`src-ui/components/layout/NavigationSidebar.tsx`):**
  - Acrylic frosted glass rail (`#12141F`, `backdrop-filter: blur(16px)`).
  - Renders the primary navigation list with active indicators, Lucide icons, and engine connectivity status.
* **`ContentViewport` (`src-ui/components/layout/ContentViewport.tsx`):**
  - Dynamic content outlet wrapped in a React Error Boundary (`ViewErrorBoundary`).
  - Provides uniform padding, scroll management (`overflow-y: overlay`), and view transition animations.

---

## 2. Ephemeral Process Lifecycle & Memory Guarantee

In accordance with `KW-NG-001`, `ADR-0002`, and `DEC-UI-01`:
* **Zero Persistent Footprint:** Clicking the close button (`✕`) triggers an immediate graceful shutdown of `kinetiwall-ui.exe`. The window intercepts the close event, dispatches any pending uncommitted drafts to Core via IPC, and calls `ExitProcess(0)`.
* **Zero Impact on Active Wallpaper Playback:** Closing or reopening the UI process **never resets, pauses, or interrupts** active video or web wallpapers running on connected displays.
* **Startup Performance Budget:** `kinetiwall-ui.exe` must mount the `AppShell`, establish its IPC channel to Core, and render the initial Library view within **$\le 350\text{ ms}$** cold launch latency.

---

## 3. Native Rust Core Boundary & IPC Integration

The application shell operates strictly within the presentation boundary. It interacts with the backend solely via typed Tauri v2 command invokes:

```typescript
// IPC Service Contract for Application Shell Initialization
export interface ShellBootstrapData {
  engineVersion: string;
  isCoreRunning: boolean;
  activeDisplaysCount: number;
  totalWallpapersCount: number;
  activePlaybackState: 'playing' | 'paused' | 'stopped';
}

export async function fetchShellBootstrap(): Promise<ShellBootstrapData> {
  return await invoke<ShellBootstrapData>('get_shell_bootstrap_state');
}
```

* **No Direct Win32 Invocations:** The shell does not interact with `WorkerW`, `Progman`, or `SetParent`. All desktop hosting actions are delegated to `kinetiwall-core.exe` over Named Pipes.
* **Single Event Listener Attachment:** Subscriptions to Core broadcasts (e.g. `topology-changed`, `playback-state-changed`) are initialized once in the root `AppShell` mount lifecycle and cleaned up cleanly on unmount.

---

## 4. Error Boundaries & View Isolation

A fatal error or unhandled JavaScript exception within an active view (e.g. malformed metadata in a third-party wallpaper package) must never crash the entire application shell or corrupt navigation:

```
[Exception Thrown in LibraryView]
               |
               v Caught by React Error Boundary
[ViewErrorBoundary Container in ContentViewport]
               |
               v Renders In-Viewport Fallback Card
+-------------------------------------------------------+
|  ⚠️ Failed to render Wallpaper Library                 |
|  Error: Corrupted package manifest in 'neon_grid.zip' |
|  [ Reload View ]              [ Report Bug ]           |
+-------------------------------------------------------+
```

* **Navigation Remains Fully Functional:** The sidebar remains interactive, allowing the user to switch to Settings, Displays, or Performance without restarting the application.

