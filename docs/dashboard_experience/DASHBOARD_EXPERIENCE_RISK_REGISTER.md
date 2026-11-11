# Dashboard Experience Risk Register & Architectural Decisions
**Document ID:** `KW-DSH-005`  
**Deliverable:** E (Dashboard Experience Risk Register)  
**Parent Document:** [README.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/dashboard_experience/README.md)  
**Classification:** Core System Architecture / Operational Reliability Standard  

---

## 1. Dashboard Experience Risk Register

Presenting real-time system state and managing quick action controls introduces distinct frontend and operational risks:

| Risk ID | Risk Summary & Failure Mode | Prob. | Impact | Severity | Architecture Mitigation & Prevention Strategy |
| :--- | :--- | :---: | :---: | :---: | :--- |
| **`RSK-DSH-01`** | **Fabricated Metrics & False Confidence**<br>Displaying dummy 60 FPS or fake 0% CPU readings when telemetry is offline misleads users during system issues. | High | Med | **HIGH** | **Truthful Data Invariant (`DEC-DSH-01`)**: Real metrics are rendered only when telemetry heartbeats arrive; otherwise, gauges are cleanly omitted. |
| **`RSK-DSH-02`** | **Action Spam Floods IPC Queue**<br>Rapid clicking on "Pause / Resume" sends dozens of conflicting commands, inducing player deadlock. | High | Med | **HIGH** | **Action Ingress Disabling (`DEC-DSH-02`)**: Action buttons enter `isLoading = true` and disable click handlers until Core confirmation arrives. |
| **`RSK-DSH-03`** | **Deadlock on Core Initialization Hang**<br>Core daemon is slow to start; Dashboard hangs indefinitely waiting for `DashboardBootstrapDTO`. | Med | High | **HIGH** | **Timeout Fallback Banner (`DEC-DSH-03`)**: Bootstrap query has a 3-second timeout, falling back to "Connecting to Core Daemon..." banner. |
| **`RSK-DSH-04`** | **Missing Thumbnail Crashes Hero Panel**<br>Corrupted or moved thumbnail image triggers unhandled image load exception, breaking Dashboard. | Med | Low | **MEDIUM** | **Graceful SVG Fallback Placeholder (`DEC-DSH-04`)**: `<img>` `onError` handler falls back to an embedded neon grid SVG placeholder. |
| **`RSK-DSH-05`** | **Layout Shift on Data Arrival**<br>Hero panel dimensions change when wallpaper details load, causing awkward visual layout jumping. | High | Low | **MEDIUM** | **Fixed Aspect Ratio Containers (`DEC-DSH-05`)**: Preview thumbnail container enforces fixed 16:9 aspect ratio (`aspect-video`) with dark skeleton. |
| **`RSK-DSH-06`** | **Missing Asset Stalls Playback Control**<br>User deleted active wallpaper file; clicking Play causes unhandled panic in player process. | Med | High | **HIGH** | **Faulted Slot Guidance (`DEC-DSH-06`)**: Core reports `AssetNotFound`; Dashboard displays "Select New Wallpaper" CTA instead of Play button. |
| **`RSK-DSH-07`** | **Continuous Polling CPU Drain**<br>Dashboard runs a 100ms `setInterval` loop to query status, elevating idle CPU overhead above 1.0%. | High | Med | **HIGH** | **Passive Event-Driven Model (`DEC-DSH-07`)**: Single bootstrap query on mount; updates pushed via passive Tauri IPC events (`ipc://status-changed`). |
| **`RSK-DSH-08`** | **XSS via Malicious Package Title**<br>Downloaded wallpaper package contains `<script>` injection in title, executing code in webview. | Med | High | **CRITICAL** | **Plain-Text Escaped Rendering (`DEC-DSH-08`)**: All metadata strings render strictly as plain text nodes; zero `dangerouslySetInnerHTML`. |

---

## 2. Architecture Decision Records (Dashboard: `DEC-DSH-01` to `08`)

The following formal decisions govern all Dashboard experience implementations across KinetiWall:

### `DEC-DSH-01`: Absolute Prohibition of Synthetic / Dummy Metrics
* **Context:** Developers often insert dummy telemetry (e.g. "60 FPS", "12% CPU") to make UI mockups look populated.
* **Decision:** Only authoritatively queried telemetry from `kinetiwall-core.exe` is displayed. If telemetry is offline, metrics are hidden.
* **Consequences:** Eliminates user confusion; ensures the dashboard remains an authoritative diagnostic surface.

### `DEC-DSH-02`: Ingress Disabling with Optimistic Loading on Quick Actions
* **Context:** Video decoders and IPC handshakes take 50–150 ms; rapid button clicks cause race conditions.
* **Decision:** Quick action buttons disable immediately upon click and render spinners until Core emits state change acknowledgments.
* **Consequences:** Completely prevents duplicate IPC submissions and player state desynchronization.

### `DEC-DSH-03`: Bounded 3-Second Bootstrap Timeout with Fallback Banner
* **Context:** The supervisor daemon may be delayed during cold boot or system startup.
* **Decision:** `fetchDashboardBootstrap()` enforces a 3000 ms timeout. If exceeded, the Dashboard enters `ENGINE_DISCONNECTED` state and retries asynchronously.
* **Consequences:** The UI remains responsive and provides immediate feedback to the user rather than freezing.

### `DEC-DSH-04`: Fixed 16:9 Aspect Ratio Hero Containers with Skeleton Loading
* **Context:** Asynchronous image loading causes cumulative layout shifts (CLS) when thumbnails arrive.
* **Decision:** Thumbnail containers enforce `aspect-video` with a dark acrylic skeleton loader.
* **Consequences:** Guarantees zero layout jumping during image loading.

### `DEC-DSH-05`: Faulted Display Slot Guidance for Missing Files
* **Context:** Users frequently move, rename, or delete wallpaper video files on local disk.
* **Decision:** When Core detects a missing file, the Dashboard suppresses playback controls and renders a targeted "Select New Wallpaper" action.
* **Consequences:** Prevents unhandled decoder crashes and guides the user directly to resolution.

### `DEC-DSH-06`: Passive Event-Driven Updates (Zero Polling)
* **Context:** Polling wastes CPU cycles and drains laptop battery.
* **Decision:** The Dashboard executes one query on mount; all subsequent updates arrive via passive event pushes (`ipc://status-changed`).
* **Consequences:** Keeps idle background CPU overhead $\le 0.05\%$, satisfying `KW-NF-001`.

### `DEC-DSH-07`: Unassigned Onboarding Experience for Empty Libraries
* **Context:** First-time users need clear guidance on how to begin using KinetiWall.
* **Decision:** When `activeWallpaper === null`, render an onboarding welcome card with a direct link to the Library view.
* **Consequences:** Creates an intuitive, self-guided onboarding flow without cluttering the screen with empty cards.

### `DEC-DSH-08`: Strict Escaped Text Rendering for Package Metadata
* **Context:** Third-party wallpaper packages are untrusted user content.
* **Decision:** All titles, descriptions, and author names render as React escaped text nodes (`{activeWallpaper.title}`).
* **Consequences:** Eliminates XSS injection risks in the desktop webview (`DEC-SEC-05`).

---

## 3. Open Engineering Review Items

| Item ID | Description | Current Architectural Status | Target Phase |
| :--- | :--- | :--- | :---: |
| **`REV-DSH-01`** | **Miniature Interactive Wallpaper Preview:** Evaluate whether the Hero panel should render a live 10 FPS webview preview or static thumbnail. | Static thumbnail approved for V1.0 to preserve $0\text{ MB}$ UI CPU footprint; live preview evaluated for V2.0. | Phase 5 (UI Implementation) |
| **`REV-DSH-02`** | **Recent Wallpapers History Carousel:** Determine whether a "Recently Played" horizontal carousel should be added below the Hero card. | Flat quick-actions approved for V1.0; history carousel scheduled for V1.1. | Phase 5 (UI Implementation) |
| **`REV-DSH-03`** | **Dynamic Color Extraction from Active Wallpaper:** Assess extracting dominant palette colors from active wallpaper to tint Hero neon glow. | Fixed Neon Cyan glow used in V1.0; dynamic canvas palette extraction evaluated for V1.2. | Phase 5 (UI Implementation) |

