# UI Component Architecture Specification
**Document ID:** `KW-UI-001`  
**Deliverable:** A (UI Component Architecture)  
**Parent Document:** [README.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/ui/README.md)  
**Classification:** Frontend Architecture / Design System Standard  

---

## 1. Five-Tier Component Hierarchy

KinetiWall organizes its user interface into five distinct architectural tiers, enforcing a unidirectional dependency flow from foundational primitives to top-level feature screens:

```
+---------------------------------------------------------------------------------+
| TIER 5: Feature Screens & Pages (`src/views/`)                                  |
|   - LibraryView, DisplayAssignmentView, PerformanceDashboardView, SettingsView  |
+---------------------------------------------------------------------------------+
                                       |
                                       v Composed Of
+---------------------------------------------------------------------------------+
| TIER 4: Domain-Specific Application Components (`src/components/domain/`)       |
|   - WallpaperCard, MonitorSelector, PlaybackControlGroup, EngineStatusBadge     |
+---------------------------------------------------------------------------------+
                                       |
                                       v Composed Of
+---------------------------------------------------------------------------------+
| TIER 3: Composed Interface Patterns (`src/components/composed/`)               |
|   - LabeledInputField, SearchBar, ConfirmationDialog, DropdownMenu, FormSection |
+---------------------------------------------------------------------------------+
                                       |
                                       v Composed Of
+---------------------------------------------------------------------------------+
| TIER 2: Reusable UI Primitives (`src/components/primitives/`)                   |
|   - Button, TextInput, Checkbox, Switch, Slider, Badge, Tooltip, Modal, Spinner |
+---------------------------------------------------------------------------------+
                                       |
                                       v Styled By
+---------------------------------------------------------------------------------+
| TIER 1: Foundations & Design Tokens (`src/styles/` & `tailwind.config.js`)      |
|   - Neon Glass Color Palette, Spacing Scale, Typography Tokens, Acrylic Blurs  |
+---------------------------------------------------------------------------------+
```

---

## 2. Ephemeral Process Lifecycle & State Ownership

A critical architectural constraint in KinetiWall is the **Ephemeral UI Lifecycle** (`KW-NG-001`, `ADR-0002`):
* `kinetiwall-ui.exe` is launched on-demand by the user or tray context menu.
* When the user clicks the window close button (`X`), the UI process calls `ExitProcess(0)` and terminates completely, dropping resident RAM consumption to **$0.0\text{ MB}$**.
* The Core supervisor daemon (`kinetiwall-core.exe`) remains running in the background.

```
[User Closes Window] ---> [Flush In-Memory Drafts to Core via IPC] ---> [ExitProcess(0)]
                                                                               |
                                                                               v
                                                                      Resident RAM: 0.0 MB
```

### 2.1 State Ownership Rules
To ensure data is not lost when `kinetiwall-ui.exe` exits:
1. **Authoritative State Lives in Core:** Display topology, active wallpaper assignments, playback states, and saved user preferences are owned by `kinetiwall-core.exe` and persisted in `config.json`.
2. **Transient UI State in Zustand:** Ephemeral client state (search query strings, active view tabs, modal open/close states, slider preview values) is managed via `Zustand` (`src/stores/`).
3. **Component Local State (`useState`):** Strictly reserved for self-contained visual interactions (e.g. tooltip visibility, hover animations, dropdown expansion).

---

## 3. Technology Stack & Directory Conventions

The frontend presentation layer is built upon the approved technology stack from Step 11 (`DEPENDENCY_INVENTORY.md`):
* **Host Framework:** Tauri v2 (`crates.io:tauri` v2.0)
* **View Library:** React 18 (`npm:react` ^18.3)
* **Language:** TypeScript 5 (`npm:typescript` ^5.5) with `strict: true`
* **Build Bundler:** Vite (`npm:vite` ^5.4)
* **Styling Engine:** Tailwind CSS (`npm:tailwindcss` ^3.4) with custom Neon Glass theme tokens
* **Client State:** Zustand (`npm:zustand` ^4.5)
* **Icons:** Lucide React (`npm:lucide-react` ^0.400)

### 3.1 Directory Organization

```
src-ui/
├── assets/                  # Bundled SVGs, brand logos, font assets
├── components/
│   ├── primitives/          # Tier 2: Button, Input, Switch, Slider, Badge
│   ├── composed/            # Tier 3: SearchBar, Dialog, Dropdown, FormField
│   └── domain/              # Tier 4: WallpaperCard, MonitorSelector, PlaybackBar
├── hooks/                   # Custom hooks (useIPC, useWallpaper, useMonitors)
├── services/                # Typed IPC bridge layer (@tauri-apps/api/core)
├── stores/                  # Zustand global client state stores
├── styles/                  # Tailwind base layers, token definitions, animations
├── types/                   # TypeScript interfaces (IPC messages, DTOs, tokens)
├── views/                   # Tier 5: Library, Monitors, Settings, Performance
└── App.tsx                  # Root shell container with navigation layout
```

---

## 4. Application Integration & IPC Boundaries

Reusable components **must never execute raw native operations or invoke shell commands directly**:

```
[Visual Component (Button)]
             |
             v Triggers Typed Event Callback
[Custom Hook: useWallpaperActions()]
             |
             v Invokes Tauri Command Bridge
[services/ipc.ts: invoke("apply_wallpaper", { displayId, wallpaperId })]
             |
             v High-Speed Named Pipe IPC
[kinetiwall-core.exe (Supervisor)]
             |
             v Native Win32 Subprocess Management
[kinetiwall-player-video.exe (Active Display)]
```

* **Contract Validation:** All parameters passed to IPC hooks are validated by TypeScript interfaces before dispatch.
* **Non-Blocking Feedback:** When a component triggers an asynchronous IPC command (e.g. applying a 4K video wallpaper), the button or card instantly transitions to a `loading` state, rendering a spinner without blocking user interaction or thread execution.
* **Error Containment:** IPC failures return typed `PlatformError` responses, which are caught and displayed via local alert banners or toast notifications without unhandled promise rejections.

