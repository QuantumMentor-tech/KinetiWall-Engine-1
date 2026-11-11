# Display Coordinate Geometry & Per-Monitor DPI Coordination
**Document ID:** `KW-DSP-003`  
**Deliverable:** C (Coordinate Geometry and DPI Specification)  
**Parent Document:** [README.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/display_management/README.md)  
**Classification:** Reference Specification / Geometry & DPI Mathematics  

---

## 1. Virtual Desktop Coordinate Systems & Spaces

Geometry math across multi-monitor Windows desktops requires rigorous distinction between three distinct coordinate spaces:

```
+---------------------------------------------------------------------------------+
| SPACE 1: Virtual Desktop Space (Win32 Global Screen Coordinates)                |
| - Origin (0,0): Top-left corner of the Primary Display                          |
| - Negative Coordinates: Displays placed to the left (X < 0) or above (Y < 0)    |
| - Total Bounding Box: [SM_XVIRTUALSCREEN, SM_YVIRTUALSCREEN, Width, Height]     |
+---------------------------------------------------------------------------------+
                                       |
                                       | Coordinate Normalization
                                       v
+---------------------------------------------------------------------------------+
| SPACE 2: WorkerW Client Space (Child Window Coordinates)                        |
| - Origin (0,0): Top-left corner of the desktop WorkerW window                   |
| - Guarantee: Coordinates are strictly NON-NEGATIVE (X >= 0, Y >= 0)             |
| - Formula: X_workerw = X_virtual - SM_XVIRTUALSCREEN                           |
|            Y_workerw = Y_virtual - SM_YVIRTUALSCREEN                           |
+---------------------------------------------------------------------------------+
                                       |
                                       | DPI Scaling / Physical Mapping
                                       v
+---------------------------------------------------------------------------------+
| SPACE 3: Hardware Device Pixel Space (Direct3D 11 Render Target)                |
| - 1:1 Pixel Mapping for SwapChains (e.g. 3840 x 2160 pixels)                    |
| - Unaffected by OS DPI virtualization under Per-Monitor V2 awareness           |
+---------------------------------------------------------------------------------+
```

---

## 2. Mathematical Coordinate Transformations

### 2.1 Negative Coordinate Normalization Example

Consider a dual-monitor setup where a 1080p secondary monitor is positioned to the left of a 4K primary monitor:
* **Primary Monitor (4K):** Rect = $[0, 0, 3840, 2160]$
* **Secondary Monitor (FHD):** Rect = $[-1920, 0, 0, 1080]$
* **Virtual Screen Origin:** $\text{SM\_XVIRTUALSCREEN} = -1920, \quad \text{SM\_YVIRTUALSCREEN} = 0$
* **Virtual Screen Extents:** Width = $5760$, Height = $2160$

Applying coordinate normalization to position child render windows inside `WorkerW`:
* **Secondary Child HWND Position:**
  $$X_{\text{child}} = -1920 - (-1920) = 0$$
  $$Y_{\text{child}} = 0 - 0 = 0$$
  $$\text{Bounds} = [0, 0, 1920, 1080]$$
* **Primary Child HWND Position:**
  $$X_{\text{child}} = 0 - (-1920) = 1920$$
  $$Y_{\text{child}} = 0 - 0 = 0$$
  $$\text{Bounds} = [1920, 0, 3840, 2160]$$

Result: Both child render windows have clean, non-negative coordinates inside the parent `WorkerW` surface. Zero clipping, zero rendering offsets.

---

## 3. Checked Integer Geometry Arithmetic & Overflow Protection

To prevent integer overflows, sign truncation, or memory corruption when computing bounding boxes or scaling viewports:

```rust
pub fn calculate_workerw_bounds(
    monitor_rect: DisplayRect,
    virtual_origin_x: i32,
    virtual_origin_y: i32,
) -> Result<DisplayRect, GeometryError> {
    // Invariant: Checked subtraction to prevent integer underflow
    let norm_x = monitor_rect.left.checked_sub(virtual_origin_x)
        .ok_or(GeometryError::CoordinateUnderflow)?;
    let norm_y = monitor_rect.top.checked_sub(virtual_origin_y)
        .ok_or(GeometryError::CoordinateUnderflow)?;

    if norm_x < 0 || norm_y < 0 {
        return Err(GeometryError::NegativeWorkerWCoordinate(norm_x, norm_y));
    }

    // Bounds check against max 32k Win32 window coordinate limits
    if norm_x > 32767 || norm_y > 32767 {
        return Err(GeometryError::CoordinateOutOfBounds(norm_x, norm_y));
    }

    Ok(DisplayRect {
        left: norm_x,
        top: norm_y,
        width: monitor_rect.width,
        height: monitor_rect.height,
    })
}
```

---

## 4. Per-Monitor V2 DPI Scaling Architecture

Windows supports heterogeneous scaling (e.g. 150% on primary display + 100% on secondary display).

### 4.1 DPI Awareness Manifest Configuration
All KinetiWall executables declare `DPI_AWARENESS_CONTEXT_PER_MONITOR_AWARE_V2` in their application manifests:
```xml
<application xmlns="urn:schemas-microsoft-com:asm.v3">
  <windowsSettings>
    <dpiAware xmlns="http://schemas.microsoft.com/SMI/2005/WindowsSettings">true/pm</dpiAware>
    <dpiAwareness xmlns="http://schemas.microsoft.com/SMI/2016/WindowsSettings">PerMonitorV2, PerMonitor</dpiAwareness>
  </windowsSettings>
</application>
```

### 4.2 Impact on Rendering & Windowing
* **Physical Device Pixel Presentation:** Win32 APIs (`GetMonitorInfoW`, `SetWindowPos`) operate directly in physical device pixels rather than virtualized logical coordinates.
* **DirectX 11 SwapChains:** `IDXGISwapChain` backbuffers are allocated with exact physical pixel dimensions ($3840 \times 2160$), guaranteeing 100% pixel-perfect clarity without GPU resampling or blur.
* **WebView2 HTML5 Wallpapers:** Web renderers receive the monitor's scale factor via CSS device pixel ratio (`window.devicePixelRatio = 1.5`), rendering crisp text, SVG vector art, and WebGL animations.

---

## 5. Spanning Mode Geometry & Gap Representation

When spanning a single wallpaper across multiple monitors, displays rarely form a perfect rectangle:

```
  (-1920, 0)                   (0, 0)
       +--------------------------+----------------------------+
       | Display 2 (1080p, 60Hz)  | Display 1 (4K UHD, 144Hz)  |
       | 1920 x 1080              | 3840 x 2160                |
       +--------------------------+                            |
       | GAP (No Physical Glass)  |                            |
       | Height: 1080px           |                            |
       +--------------------------+----------------------------+
```

### 5.1 Spanning Union Geometry Invariants
1. **Union Bounding Box:** Spanning player creates a single render surface equal to the bounding union of all included displays ($5760 \times 2160$).
2. **Virtual Void Handling:** Regions where no physical monitor exists (the "GAP" in the diagram above) are left unpresented; DWM composition naturally discards frames in non-display coordinates without consuming memory or fill-rate bandwidth.
3. **Bezel Compensation (`V1.1` Deferred):** Calculating physical monitor bezel pixel widths to prevent visual misalignment across physical bezels is deferred to V1.1.

