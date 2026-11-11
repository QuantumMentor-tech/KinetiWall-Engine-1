# Display Management Architecture Specification
**Document ID:** `KW-DSP-001`  
**Deliverable:** A (Display Management Architecture)  
**Parent Document:** [README.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/display_management/README.md)  
**Classification:** Windows Display Systems & Multi-Monitor Standard  

---

## 1. Subsystem Responsibilities & Boundary Architecture

Display management in KinetiWall operates under a strict single-writer supervisor model:

```
+---------------------------------------------------------------------------------+
|                        kinetiwall-core.exe (Supervisor)                         |
|                                                                                 |
|  +---------------------------------------------------------------------------+  |
|  |                   Display Topology Manager (Authoritative)                |  |
|  | - Listens to WM_DISPLAYCHANGE via Hidden Window                           |  |
|  | - Queries EnumDisplayMonitors & GetMonitorInfoW                           |  |
|  | - Maintains Monotonic Topology Generation ID (TopologyGenerationId)       |  |
|  | - Resolves Persistent Wallpaper Assignments from config.json              |  |
|  | - Calculates Normalized Window Bounds for WorkerW Child HWNDs             |  |
|  +---------------------------------------------------------------------------+  |
+---------------------------------------------------------------------------------+
          |                                               |
          | IPC Topology Broadcast                        | IPC Topology Query
          v                                               v
+-------------------------------+              +----------------------------------+
| kinetiwall-player-video.exe   |              | kinetiwall-ui.exe                |
| Display: \\.\DISPLAY1         |              | (Ephemeral Settings GUI)         |
| - Read-only display bounds    |              | - Renders display layout preview |
| - DXGI SwapChain sizing       |              | - User changes wallpaper targets |
+-------------------------------+              +----------------------------------+
```

### 1.1 Responsibility Allocation

* **`kinetiwall-core.exe` (Display Supervisor):**
  - Sole component authorized to register for OS display notifications (`WM_DISPLAYCHANGE`).
  - Queries Win32 enumeration APIs and constructs the authoritative `TopologySnapshot`.
  - Maps connected monitors to saved user preferences in `config.json`.
  - Issues spawn, resize, and terminate commands to per-display player processes.
* **`kinetiwall-player-*.exe` (Per-Display Renderers):**
  - Receives assigned monitor bounds from Core via IPC (`CMD_SET_DISPLAY_GEOMETRY`).
  - Resizes Direct3D 11 swapchain buffers (`IDXGISwapChain::ResizeBuffers`) or WebView2 viewports.
  - Zero authority to reconfigure display topology or query global monitor lists independently.
* **`kinetiwall-ui.exe` (Ephemeral Management GUI):**
  - Queries `TopologySnapshot` from Core via IPC (`REQ_GET_DISPLAYS`).
  - Visualizes monitor topology in the Settings UI with correct aspect ratios and relative offsets.
  - Submits user reassignment commands to Core; never writes to display hardware directly.

---

## 2. Canonical Display Model

To isolate domain logic from low-level Win32 types, KinetiWall represents displays through an immutable, canonical Rust data model:

```rust
/// Canonical representation of a physical or virtual display device
#[derive(Debug, Clone, PartialEq, Eq, serde::Serialize, serde::Deserialize)]
pub struct DisplayDescriptor {
    /// Internal runtime identifier (e.g. "disp-1")
    pub id: String,
    /// Win32 GDI Device Name (e.g. "\\\\.\\DISPLAY1")
    pub gdi_device_name: String,
    /// Friendly human-readable monitor model name (e.g. "DELL U2720Q")
    pub friendly_name: String,
    /// Cryptographic SHA-256 hash of hardware EDID block (Persistent Key)
    pub hardware_edid_hash: String,
    /// True if this display is the Windows primary monitor (origin 0,0)
    pub is_primary: bool,
    /// Physical monitor bounding box in virtual screen coordinates
    pub physical_bounds: DisplayRect,
    /// Normalized coordinates relative to desktop WorkerW client area
    pub workerw_bounds: DisplayRect,
    /// Effective DPI scaling factor (e.g. 144 for 150% scaling, baseline 96)
    pub dpi: u32,
    /// Vertical refresh rate in millihertz (e.g. 59940 for 59.94Hz, 144000 for 144Hz)
    pub refresh_rate_mhz: u32,
    /// Current display orientation
    pub orientation: DisplayOrientation,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq, serde::Serialize, serde::Deserialize)]
pub struct DisplayRect {
    pub left: i32,
    pub top: i32,
    pub width: u32,
    pub height: u32,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq, serde::Serialize, serde::Deserialize)]
pub enum DisplayOrientation {
    Landscape,
    Portrait,
    LandscapeFlipped,
    PortraitFlipped,
}
```

---

## 3. Windows Display Enumeration Strategy

Display discovery utilizes the standard, documented Win32 GDI monitor enumeration pipeline:

```
[Trigger: Engine Startup or WM_DISPLAYCHANGE]
                      |
                      v
       EnumDisplayMonitors(NULL, NULL, ...)
                      |
        +-------------+-------------+
        |                           |
  [For Each HMONITOR]       [Query Virtual Screen Metrics]
        |                           |
GetMonitorInfoW(hMon)       GetSystemMetrics(SM_XVIRTUALSCREEN)
Query MONITORINFOEXW        GetSystemMetrics(SM_YVIRTUALSCREEN)
        |                           |
Query EDID via WMI / SetupAPI       |
Compute SHA-256 Hash                |
        |                           |
        +-------------+-------------+
                      |
                      v
         Construct Canonical TopologySnapshot
```

### 3.1 Authoritative Native APIs

1. **`EnumDisplayMonitors` (User32):** Enumerates all display monitors intersecting a clipping region. Passing `NULL` enumerates all monitors visible to the virtual desktop.
2. **`GetMonitorInfoW` (User32):** Retrieves `MONITORINFOEXW`, providing:
   * `rcMonitor`: Physical screen coordinates in virtual desktop space.
   * `dwFlags`: Checked for `MONITORINFOF_PRIMARY` to determine primary monitor status.
   * `szDevice`: GDI adapter name (e.g. `\\.\DISPLAY1`).
3. **`GetDpiForMonitor` (Shcore) / `GetDpiForWindow` (User32):** Queries per-monitor DPI scaling under `DPI_AWARENESS_CONTEXT_PER_MONITOR_AWARE_V2`.
4. **EDID Query (SetupAPI / Display Devices):** Reads the 128-byte hardware EDID descriptor from the Windows Registry device key (`Enum\DISPLAY\...`) to extract the manufacturer ID, product code, and serial number.

---

## 4. Multi-Monitor Rendering Modes

KinetiWall supports three distinct multi-monitor rendering modes:

### 4.1 Mode 1: Independent Wallpaper Mode (`V1.0` Core Feature)
* **Behavior:** Every active display runs a distinct, independently assigned wallpaper (e.g. 4K video wallpaper on Display 1, interactive web wallpaper on Display 2).
* **Process Topology:** In conformance with `ADR-0002` and `PROC-01..04`, each active display is assigned a dedicated child player process (`kinetiwall-player-video.exe` or `kinetiwall-player-web.exe`).
* **Failure Isolation:** If Player 1 crashes, Core isolates the failure, leaves Player 2 running uninterrupted, and applies the circuit-breaker recovery policy to Player 1 (`DEC-RCV-01`).

### 4.2 Mode 2: Spanning Wallpaper Mode (`V1.0` Core Feature)
* **Behavior:** A single high-resolution wallpaper (e.g. ultra-wide $5120 \times 1440$ video or panoramic 3D canvas) spans seamlessly across multiple adjacent monitors.
* **Process Topology:** A single dedicated player process owns the spanning presentation surface.
* **Geometry Calculation:** Core computes the **bounding union rectangle** covering all participant displays:
  $$\text{UnionLeft} = \min_{i}(\text{Rect}_i.\text{left}), \quad \text{UnionTop} = \min_{i}(\text{Rect}_i.\text{top})$$
  $$\text{UnionRight} = \max_{i}(\text{Rect}_i.\text{right}), \quad \text{UnionBottom} = \max_{i}(\text{Rect}_i.\text{bottom})$$
* **Gaps & Disjoint Monitors:** If displays have differing heights or are non-contiguous, the spanning player renders into the bounding union; regions without physical glass are clipped by Windows composition.

### 4.3 Mode 3: Duplicated / Clone Presentation Mode (`V1.1` Deferred Feature)
* **Behavior:** Identical wallpaper frames are mirrored across secondary displays (e.g. presentation monitors or capture cards).
* **Architecture Implementation:** Implemented inside KinetiWall by sharing DirectX 11 textures via NT shared handles (`IDXGIResource1::CreateSharedHandle`), rendering identical frames across multiple swapchains.
* **Non-Goal Guardrail:** KinetiWall **never** alters the Windows OS display topology to duplicate displays (`KW-NG-004`). OS-level cloning is the responsibility of Windows display settings.

