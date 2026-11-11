# Responsive Desktop Layout & Window Sizing Specification
**Document ID:** `KW-SHL-003`  
**Deliverable:** C (Responsive Desktop Layout)  
**Parent Document:** [README.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/application_shell/README.md)  
**Classification:** Frontend Layout & Responsive Design Standard  

---

## 1. Desktop Breakpoints & Window Size Constellation

Unlike mobile web applications that collapse into hamburger menus, KinetiWall is an administrative desktop interface for Microsoft Windows. Its responsive behavior is engineered around common Windows desktop window sizes, Windows 11 Snap Layouts, and multi-monitor scaling:

```
+---------------------------------------------------------------------------------------------------+
| BREAKPOINT TIERS                                                                                  |
+---------------------------------------------------------------------------------------------------+
| 1. COMPACT WINDOW (Min Bounds):   960px  <= Width < 1200px   (e.g. Half-screen 1080p Snap Layout) |
| 2. STANDARD WINDOW (Default):     1200px <= Width < 1600px   (e.g. Standard centered desktop view)|
| 3. WIDE WINDOW (Maximized 1440p): 1600px <= Width < 2400px   (e.g. Maximized QHD / Fullscreen)    |
| 4. ULTRA-WIDE (4K UHD / Spanning):Width >= 2400px             (e.g. Maximized 4K / 32:9 displays)  |
+---------------------------------------------------------------------------------------------------+
```

### 1.1 Responsive Layout Matrix across Breakpoints

| UI Region / Feature | Compact Window ($960\text{--}1199\text{ px}$) | Standard Window ($1200\text{--}1599\text{ px}$) | Wide Window ($1600\text{--}2399\text{ px}$) | Ultra-Wide ($\ge 2400\text{ px}$) |
| :--- | :--- | :--- | :--- | :--- |
| **Navigation Sidebar** | Icon + Compact Label ($200\text{ px}$) | Standard Sidebar ($240\text{ px}$) | Standard Sidebar ($260\text{ px}$) | Standard Sidebar ($280\text{ px}$) |
| **Library Grid Columns** | **2 Columns** (Cards $340\text{ px}$) | **3 Columns** (Cards $300\text{ px}$) | **4 Columns** (Cards $320\text{ px}$) | **6 Columns** (Cards $340\text{ px}$) |
| **Monitor Canvas** | Stacked / Scaled ($0.6\times$) | Scaled ($0.8\times$) | Full Size ($1.0\times$) | Expanded Canvas ($1.2\times$) |
| **Settings Forms** | Single-column form rows | Two-column grid layout | Two-column grid layout | Three-column grouped layout |
| **Header Bar Controls** | Condensed status badges | Full engine metrics & tray icons | Expanded diagnostic meters | Expanded diagnostic meters |

---

## 2. Window Minimum Constraints & Windows Snap Compatibility

In accordance with Windows platform guidelines, `kinetiwall-ui.exe` configures its native Tauri window limits:

```json
{
  "tauri": {
    "windows": [
      {
        "title": "KinetiWall Engine",
        "width": 1280,
        "height": 768,
        "minWidth": 960,
        "minHeight": 600,
        "resizable": true,
        "decorations": false,
        "transparent": true
      }
    ]
  }
}
```

* **Minimum Usable Bounds ($960 \times 600\text{ px}$):** Guarantees that users snapping KinetiWall to a 50% split on a 1080p display ($960\text{ px}$ half-width) experience zero visual clipping, horizontal scrollbars, or overlapping controls.
* **Windows 11 Snap Layout Integration:** The custom `WindowHeaderBar` supports native Windows 11 Snap Assist menus when hovering over the Maximize button (`tauri-plugin-decorum` / `WM_NCHITTEST` `HTMAXBUTTON`).

---

## 3. High-DPI Scaling & Device Pixel Ratio (DPR)

Modern Windows setups combine high-DPI displays (e.g. Surface laptop at 200% scaling, 4K monitor at 150% scaling, secondary display at 100% scaling).

* **Vector Asset Scaling:** All icons in the shell use vector SVGs (Lucide React), scaling with mathematical precision without pixelation or blurry edges.
* **Font Scaling:** Typography utilizes relative `rem` units based on a normalized 16px baseline (`1rem = 16px`), adapting cleanly to Windows system font scaling preferences.
* **Hardware Device Pixel Ratio:** Canvas previews (e.g. MonitorSelector topology layout) multiply their internal buffer dimensions by `window.devicePixelRatio` to prevent blurry lines on Retina/4K screens:
  $$\text{CanvasWidth} = \text{ClientWidth} \times \text{window.devicePixelRatio}$$
  $$\text{CanvasHeight} = \text{ClientHeight} \times \text{window.devicePixelRatio}$$

