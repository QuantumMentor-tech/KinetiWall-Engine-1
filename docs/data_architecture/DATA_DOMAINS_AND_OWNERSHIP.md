# DATA DOMAIN INVENTORY & OWNERSHIP SPECIFICATION
## Document ID: KW-DAT-001 | Phase 2: System Architecture & Technical Decisions

---

### 1. Architectural Scope and Principles

This specification defines all data domains managed by KinetiWall across its multi-process lifecycle. Data domains are categorized into **Persistent** (must survive application restarts, power loss, and engine updates) and **Transient** (ephemeral or fully reconstructible caches).

To enforce zero-data-corruption and prevent multi-process file locking hazards on Windows, every data domain is assigned a single **Authoritative Owner Component** within the architecture defined in **Step 8 (KW-PRC-001..005)**.

---

### 2. Comprehensive Data Domain Inventory

```
+--------------------------------------------------------------------------------------------------------------------------+
|                                                  KINETIWALL DATA DOMAINS                                                 |
+-------------------------------------------------+------------------------------------------------------------------------+
|                   PERSISTENT                    |                               TRANSIENT                                |
|  - Application Settings (config.json)           |  - Active Playback State (RAM)                                         |
|  - Monitor & Display Topology (config.json)     |  - Decoded Frame Buffers & GPU Textures (VRAM)                         |
|  - Per-Wallpaper Customizations (config.json)   |  - IPC Messages & Pipes (Named Pipe Buffers)                           |
|  - Package Manifests (project.json)             |  - Render Cache & Compiled Shaders (Disk Cache)                       |
|  - Wallpaper Asset Files (Filesystem)           |  - Thumbnail & Preview Cache (Disk Cache)                              |
|  - Migration & Schema State (config.json)       |  - Operational & Crash Diagnostics (Log Files)                        |
+-------------------------------------------------+------------------------------------------------------------------------+
```

#### 2.1 Domain 1: Wallpaper Library Metadata (Persistent)
- **Description**: High-level descriptive metadata for all installed wallpapers (title, description, tags, author, license, content hash, install timestamp, content rating).
- **Authoritative Owner**: `LibraryRepository` (inside `kinetiwall-core.exe`).
- **Allowed Writers**: `LibraryRepository` exclusively.
- **Allowed Readers**: `kinetiwall-core.exe` (internal), `kinetiwall-ui.exe` (via IPC read queries).
- **Physical Storage Backend**: Stored within individual `project.json` package manifests inside `%LOCALAPPDATA%\KinetiWall\wallpapers\{wallpaper_id}\project.json`, indexed into an in-memory `LibraryIndex` cache upon core startup.
- **Persistence Duration**: Indefinite (lives until the wallpaper package is explicitly deleted by the user).
- **Recovery Tier**: **Critical**. Loss means wallpaper package identity and descriptive information are lost.

#### 2.2 Domain 2: Wallpaper Asset Files (Persistent)
- **Description**: Raw binary media files (MP4, MKV, WebM), HTML5 web application bundles (HTML, JS, CSS, WebGL shaders), static images, and sound effects.
- **Authoritative Owner**: `PackageManager` / `AssetStore` (inside `kinetiwall-core.exe`).
- **Allowed Writers**: `PackageManager` exclusively during package import or package deletion.
- **Allowed Readers**: Player subprocesses (`kinetiwall-player-video.exe`, `kinetiwall-player-web.exe`) via direct read-only filesystem handles; `kinetiwall-core.exe` for hashing and integrity validation.
- **Physical Storage Backend**: Local directory hierarchy `%LOCALAPPDATA%\KinetiWall\wallpapers\{wallpaper_id}\assets\`.
- **Persistence Duration**: Indefinite (tied to wallpaper package lifecycle).
- **Recovery Tier**: **Critical**. User-installed content cannot be reconstructed if deleted from the library.

#### 2.3 Domain 3: Application Settings (Persistent)
- **Description**: Global engine configuration including autostart preference, pause/play performance rules (fullscreen, battery, remote desktop), default audio routing, target framerate caps, theme preferences, and hardware acceleration toggles.
- **Authoritative Owner**: `ConfigurationRepository` (inside `kinetiwall-core.exe`).
- **Allowed Writers**: `ConfigurationRepository` exclusively.
- **Allowed Readers**: All engine components in `kinetiwall-core.exe`; UI reads via IPC query (`REQ_GET_CONFIG`).
- **Physical Storage Backend**: `%LOCALAPPDATA%\KinetiWall\config.json`.
- **Persistence Duration**: Indefinite.
- **Recovery Tier**: **Critical**. Defaults restored from embedded compiled baseline if corrupted, but user configuration would be lost. Backed up automatically to `config.json.bak` on every successful write.

#### 2.4 Domain 4: Display & Monitor Assignments (Persistent)
- **Description**: Current mapping of physical and virtual display devices to wallpapers, including monitor device path (`\\.\DISPLAY1`), monitor hardware EDID ID, assigned wallpaper UUID, layout stretch mode (`Fill`, `Fit`, `Stretch`, `Center`), audio mute status, and volume scale.
- **Authoritative Owner**: `MonitorManager` & `ConfigurationRepository` (inside `kinetiwall-core.exe`).
- **Allowed Writers**: `ConfigurationRepository` exclusively.
- **Allowed Readers**: `PlaybackCoordinator`, `LifecycleSupervisor`, UI frontend.
- **Physical Storage Backend**: Stored under the `monitors` key within `%LOCALAPPDATA%\KinetiWall\config.json`.
- **Persistence Duration**: Indefinite (survives display disconnect/reconnect cycles; matches by persistent EDID / hardware identifier).
- **Recovery Tier**: **Critical**.

#### 2.5 Domain 5: Per-Wallpaper Dynamic Customization Properties (Persistent)
- **Description**: User-modified dynamic runtime properties overriding manifest defaults (e.g., custom clock color `#FF0055`, particle speed `1.5`, background blur `true`, custom text string `"Cyberpunk 2077"`).
- **Authoritative Owner**: `ConfigurationRepository` (inside `kinetiwall-core.exe`).
- **Allowed Writers**: `ConfigurationRepository` exclusively upon receiving user updates from UI (`REQ_SET_PROPERTY`).
- **Allowed Readers**: `ConfigurationRepository` transmits merged property values to player subprocesses via IPC configuration sync (`MSG_SYNC_PROPERTIES`).
- **Physical Storage Backend**: Stored under the `wallpaper_properties` section in `%LOCALAPPDATA%\KinetiWall\config.json`.
- **Persistence Duration**: Indefinite. Preserved across engine restarts even if wallpaper is temporarily unassigned from all displays.
- **Recovery Tier**: **Critical**.

#### 2.6 Domain 6: Package Metadata & Capabilities (Persistent)
- **Description**: Technical declarations of package capabilities, required engine version (`min_engine_version`), target player type (`video` | `web`), requested sandboxing permissions (network access, local storage, audio capture), and dynamic property definitions.
- **Authoritative Owner**: `PackageManager` (inside `kinetiwall-core.exe`).
- **Allowed Writers**: Read-only once installed. Only modified if the wallpaper package is formally updated.
- **Allowed Readers**: `PackageManager`, `PlaybackCoordinator`, `kinetiwall-ui.exe`.
- **Physical Storage Backend**: `%LOCALAPPDATA%\KinetiWall\wallpapers\{wallpaper_id}\project.json`.
- **Persistence Duration**: Indefinite.
- **Recovery Tier**: **Critical**.

#### 2.7 Domain 7: Thumbnail & Preview Cache (Transient / Reconstructible)
- **Description**: Pre-rendered static image thumbnails (WebP / PNG, 320x180 and 640x360) and optional short looping animated previews used by the UI gallery.
- **Authoritative Owner**: `ThumbnailService` (inside `kinetiwall-core.exe`).
- **Allowed Writers**: `ThumbnailService` exclusively.
- **Allowed Readers**: `kinetiwall-ui.exe` reads directly via local URI (`file:///...`) or streamed via IPC.
- **Physical Storage Backend**: `%LOCALAPPDATA%\KinetiWall\cache\thumbnails\{wallpaper_id}_{size}.webp`.
- **Persistence Duration**: Transient. Evicted on LRU basis when total cache exceeds configured quota (default 256 MB).
- **Recovery Tier**: **Reconstructible**. Can be fully regenerated on-the-fly from the source wallpaper assets (`assets/video.mp4` or package thumbnail).

#### 2.8 Domain 8: Diagnostic & Telemetry State (Transient / Ephemeral)
- **Description**: Timestamped structured log files, crash dump minidumps (`.dmp`), performance traces, and IPC communication fault logs.
- **Authoritative Owner**: `DiagnosticsLogger` (inside `kinetiwall-core.exe`).
- **Allowed Writers**: `DiagnosticsLogger` in core; crash handler processes write directly to the crash directory upon fatal unhandled exceptions.
- **Allowed Readers**: Local support tooling or user inspection. Zero automated outbound cloud transmission.
- **Physical Storage Backend**: `%LOCALAPPDATA%\KinetiWall\logs\kinetiwall_{date}.log` and `%LOCALAPPDATA%\KinetiWall\crashes\{timestamp}.dmp`.
- **Persistence Duration**: 7 days rolling retention, max 50 MB total log directory cap.
- **Recovery Tier**: **Ephemeral**. Purely diagnostic; deletion has zero impact on engine operation.

#### 2.9 Domain 9: Migration & Versioning Metadata (Persistent)
- **Description**: Historical schema version stamps, successful migration execution logs, and rollback state flags.
- **Authoritative Owner**: `MigrationEngine` (inside `kinetiwall-core.exe`).
- **Allowed Writers**: `MigrationEngine` exclusively during engine startup schema reconciliation.
- **Allowed Readers**: `ConfigurationRepository`.
- **Physical Storage Backend**: Root fields `schema_version` and `migration_history` inside `config.json`.
- **Persistence Duration**: Indefinite.
- **Recovery Tier**: **Critical**. Dictates whether configuration can be safely parsed or requires forward transformation.

---

### 3. Data Domain Matrix & Governance Rules

| Domain ID | Data Domain Name | Authoritative Owner | Readers | Storage Path | Format | Recovery Tier | Max Size Cap |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **DOM-01** | Wallpaper Library Metadata | `LibraryRepository` | Core, UI | `wallpapers/{id}/project.json` | JSON | Critical | ~50 KB / item |
| **DOM-02** | Wallpaper Asset Files | `PackageManager` | Players, Core | `wallpapers/{id}/assets/` | MP4/HTML/PNG | Critical | User Bound |
| **DOM-03** | Application Settings | `ConfigurationRepo` | Core, UI | `config.json` | JSON | Critical | < 500 KB |
| **DOM-04** | Monitor Assignments | `MonitorManager` | Core, UI | `config.json` (`monitors`) | JSON | Critical | < 100 KB |
| **DOM-05** | Custom Properties | `ConfigurationRepo` | Core, Players | `config.json` (`properties`) | JSON | Critical | < 500 KB |
| **DOM-06** | Package Metadata | `PackageManager` | Core, UI | `wallpapers/{id}/project.json` | JSON | Critical | < 50 KB / item |
| **DOM-07** | Thumbnail Cache | `ThumbnailService` | Core, UI | `cache/thumbnails/` | WebP/PNG | Reconstructible | 256 MB Quota |
| **DOM-08** | Diagnostics & Logs | `DiagnosticsLogger` | Core, User | `logs/`, `crashes/` | Plaintext/DMP | Ephemeral | 50 MB Quota |
| **DOM-09** | Migration Metadata | `MigrationEngine` | Core | `config.json` (`schema`) | JSON | Critical | < 10 KB |

---

### 4. Concurrency Isolation and Lifetime Invariants

1. **No Shared Writer Invariant**:
   Under no circumstances shall two OS threads or processes attempt concurrent file writes to `config.json` or any `project.json`. All mutations are serialized through the Core Engine's actor queue or Tokio task pipeline.
2. **Player Read Isolation Invariant**:
   Player subprocesses (`kinetiwall-player-video.exe`, `kinetiwall-player-web.exe`) MUST NEVER open handles to `config.json`. They receive all necessary configuration parameters, audio states, and property values via strictly typed IPC payloads.
3. **Cache Purge Immunity Invariant**:
   Deleting the entire `cache/` directory while the application is stopped MUST NOT cause engine launch failure, data loss, or configuration reset. The engine must seamlessly regenerate any missing thumbnails on demand.

