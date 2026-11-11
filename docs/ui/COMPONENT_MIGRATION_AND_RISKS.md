# UI Component Migration & Risk Register
**Document ID:** `KW-UI-005`  
**Deliverable:** E (Component Migration and Preservation Report)  
**Parent Document:** [README.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/ui/README.md)  
**Classification:** Core System Architecture / Operational Reliability Standard  

---

## 1. UI Component Risk Register

Developing a high-performance desktop live wallpaper interface presents distinct technical and architectural risks:

| Risk ID | Risk Summary & Failure Mode | Prob. | Impact | Severity | Architecture Mitigation & Prevention Strategy |
| :--- | :--- | :---: | :---: | :---: | :--- |
| **`RSK-UI-01`** | **UI Process Survives Window Close**<br>Closing settings window fails to terminate `kinetiwall-ui.exe`, leaking $100\text{ MB}$ persistent RAM in background. | Med | High | **CRITICAL** | **Deterministic ExitProcess on Close (`DEC-UI-01`)**: Window `WM_CLOSE` handler flushes drafts and calls `ExitProcess(0)`, guaranteeing $0.0\text{ MB}$ closed footprint. |
| **`RSK-UI-02`** | **XSS Injection via Malicious Wallpaper Title**<br>A downloaded wallpaper package injects `<script>` tags into title, executing code in Tauri webview. | Med | High | **CRITICAL** | **Strict Escaped Rendering (`DEC-UI-02`)**: All manifest strings are rendered as text nodes; `dangerouslySetInnerHTML` is prohibited; WebView2 runs sandboxed. |
| **`RSK-UI-03`** | **Focus Trap Locks Keyboard Navigation**<br>Poorly authored modal dialog traps keyboard focus without escape path, freezing user interaction. | Med | High | **HIGH** | **Escape Protocol & Focus Restoration (`DEC-UI-03`)**: Dialogs intercept `Escape`, restore focus to trigger element, and support click-outside dismissal. |
| **`RSK-UI-04`** | **Asynchronous Action Race Conditions**<br>Rapid repeated clicks on "Apply Wallpaper" send 5 duplicate IPC commands, crashing or thrashing player. | High | Med | **HIGH** | **Instant Loading State & Debounce (`DEC-UI-04`)**: Action buttons disable immediately upon click and render spinners until IPC acknowledgments arrive. |
| **`RSK-UI-05`** | **Contrast Failure in Dark Acrylic Theme**<br>Muted text or inactive controls blend into dark background, failing WCAG 2.1 AA accessibility. | Med | Med | **MEDIUM** | **Enforced Contrast Ratios (`DEC-UI-05`)**: All text colors maintain $\ge 4.5:1$ contrast against acrylic cards; focus rings maintain $\ge 9.8:1$. |
| **`RSK-UI-06`** | **IPC Flooding via Slider Adjustments**<br>Dragging volume or speed slider emits 60 IPC messages per second, saturating Named Pipe queue. | High | Med | **HIGH** | **Client-Side Throttling (`DEC-UI-06`)**: Sliders update local visual state immediately at 60 FPS, but throttle IPC emissions to 10 Hz (every 100 ms). |
| **`RSK-UI-07`** | **Motion Sickness / Epilepsy Trigger**<br>Rapid blinking neon animations or unconstrained card movements disorient users with vestibular disorders. | Low | High | **HIGH** | **Mandatory Reduced-Motion Respect (`DEC-UI-07`)**: `@media (prefers-reduced-motion: reduce)` instantly eliminates all transitions across components. |
| **`RSK-UI-08`** | **Stale Topology in Monitor Selector**<br>User unplugs monitor while UI is open; UI displays obsolete monitor layout, causing failed assignments. | Med | Med | **MEDIUM** | **Core IPC Topology Push (`DEC-UI-08`)**: Core pushes updated `TopologySnapshot` to UI via Tauri event bridge whenever `WM_DISPLAYCHANGE` triggers. |

---

## 2. Architecture Decision Records (UI Architecture: `DEC-UI-01` to `08`)

The following formal decisions govern all reusable UI component implementations across KinetiWall:

### `DEC-UI-01`: Deterministic Process Termination on Window Close
* **Context:** Live wallpaper users demand minimal background footprint; running persistent Electron/Webview GUIs wastes RAM.
* **Decision:** Closing the `kinetiwall-ui.exe` window terminates the process completely via `ExitProcess(0)`.
* **Consequences:** Reusable components must not retain authoritative state; all persistent data is stored in Core via IPC. Guarantees $0.0\text{ MB}$ closed RAM.

### `DEC-UI-02`: Absolute Prohibition of `dangerouslySetInnerHTML`
* **Context:** Wallpaper titles, author names, and descriptions are loaded from third-party ZIP/JSON archives.
* **Decision:** Render all text using standard React text nodes (`{wallpaper.title}`). Unsanitized HTML rendering is strictly forbidden.
* **Consequences:** Eliminates cross-site scripting (XSS) and remote code execution vulnerabilities in the desktop webview.

### `DEC-UI-03`: Accessible Focus Trap with Restored Focus on Dismissal
* **Context:** Dialogs must prevent keyboard users from tabbing into hidden background layers.
* **Decision:** Modals enforce tab wrapping within the dialog, intercept `Escape` to close, and restore focus to the calling element.
* **Consequences:** Full compliance with WCAG 2.1 AA guidelines.

### `DEC-UI-04`: Instant Visual Feedback & Ingress Disabling for Actions
* **Context:** Video decoders and file unpacking take hundreds of milliseconds; duplicate clicks cause race conditions.
* **Decision:** Action buttons enter `isLoading` immediately upon click, disabling further clicks and rendering a spinner.
* **Consequences:** Prevents duplicate IPC submissions and eliminates player startup race conditions.

### `DEC-UI-05`: WCAG 2.1 Level AA Contrast Standards Across All Tokens
* **Context:** Dark cyberpunk palettes can suffer from low readability if text contrast is not strictly managed.
* **Decision:** All text tokens must achieve $\ge 4.5:1$ contrast against `#12141F`; primary action text achieves $15.2:1$.
* **Consequences:** Guarantees crisp readability in bright daylight or dark room environments.

### `DEC-UI-06`: Slider Real-Time Visuals with 10 Hz IPC Throttling
* **Context:** Dragging sliders requires 60 FPS visual smoothness, but IPC pipes should not be flooded with high-frequency messages.
* **Decision:** Update local React/Zustand state at 60 FPS; throttle Named Pipe IPC dispatches to Core to 10 Hz (every 100 ms).
* **Consequences:** Delivers butter-smooth slider interaction while keeping IPC overhead $< 0.1\%$ CPU.

### `DEC-UI-07`: Universal `@media (prefers-reduced-motion)` Respect
* **Context:** Vestibular disorders can be triggered by animated scaling or sliding transitions.
* **Decision:** Implement global CSS overrides stripping all transition durations when Windows animations are disabled.
* **Consequences:** Ensures accessibility compliance and respects user operating system preferences.

### `DEC-UI-08`: Event-Driven Topology Synchronization via Tauri IPC
* **Context:** Display topology can change at any moment via hotplugging while the settings UI is open.
* **Decision:** Core broadcasts an `ipc://topology-updated` event to the UI whenever `WM_DISPLAYCHANGE` stabilizes.
* **Consequences:** MonitorSelector updates in real-time without requiring manual user refresh.

---

## 3. Open Engineering Review Items

| Item ID | Description | Current Architectural Status | Target Phase |
| :--- | :--- | :--- | :---: |
| **`REV-UI-01`** | **Virtual List Windowing for Large Libraries:** Determine whether libraries $> 500$ items require `@tanstack/react-virtual` for DOM recycling. | Planned as enhancement if library size exceeds 200 items; DOM node limit tested in Phase 5. | Phase 5 (UI Implementation) |
| **`REV-UI-02`** | **Custom Acrylic Frosted Glass on Windows 10 vs 11:** Evaluate native Mica / Acrylic window effects (`tauri-plugin-decorum` or `DwmSetWindowAttribute`). | CSS backdrop-filter fallback used by default; native Mica titlebar evaluation scheduled for Phase 5. | Phase 5 (UI Implementation) |
| **`REV-UI-03`** | **Internationalization (i18n) Framework:** Plan localization architecture for multi-language string tables. | Prop contracts designed for string localization keys; library selection scheduled for V1.1. | Phase 5 (UI Implementation) |

