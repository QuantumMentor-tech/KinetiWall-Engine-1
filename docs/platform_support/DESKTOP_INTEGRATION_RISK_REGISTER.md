# KinetiWall — Windows API & Desktop Integration Risk Register

**Document Identifier:** KW-POL-003  
**Document Type:** Systems-Level Desktop Integration Risk Assessment (Deliverable C)  
**Current Phase:** Phase 1 — Project Research, Requirements & Feasibility  
**Step:** Step 4 of 90 — Windows Compatibility & Platform Support Policy  
**Status:** Approved Integration Risk Register  
**Date of Research:** 2026-11-08  

---

## 1. Integration Methodology & Documentation Integrity

A core premise of KinetiWall is rendering live wallpapers directly behind Windows desktop icons without interfering with shell interactions (icon clicks, selection marquees, context menus). 

Because Microsoft does not provide an officially supported, documented public API specifically designed for live wallpaper hosting, desktop customization applications rely on a mixture of **documented Win32 APIs** and **undocumented Windows Explorer shell behaviors**.

This register catalogs each technical mechanism, classifies its documentation status, assesses its operational risk across Windows releases, and prescribes concrete fallback and recovery procedures.

### Classification Taxonomy:
- **`[DOC_API]`**: Officially documented Microsoft Win32/COM API with published lifecycle and compatibility guarantees.
- **`[UNDOC_BEHAVIOR]`**: Undocumented internal Windows shell implementation detail subject to unannounced modification or breakage by Microsoft.
- **`[OBSERVED_COMMUNITY]`**: Observed working in active open-source engines (e.g. Lively Wallpaper) across specific Windows builds.
- **`[CONFIRMED_SPIKE]`**: Validated through an isolated KinetiWall prototype test on the physical operating system.
- **`[UNVERIFIED_ASSUMPTION]`**: Theoretical or reported behavior awaiting empirical validation.

---

## 2. Desktop Integration Risk Analysis Matrix

| ID | Integration Mechanism | Classification | Supported Windows Releases | Known Technical Failure Mode | Severity | Mitigation & Safe Fallback Strategy | Proof-of-Concept Target |
| :--- | :--- | :---: | :--- | :--- | :---: | :--- | :--- |
| **IR-001** | **Progman Window Discovery** (`FindWindowW(L"Progman", NULL)`) | `[DOC_API]` | All Windows NT releases (Win 7 through Win 11 24H2) | Handle is NULL during Explorer bootup or shell crashes | **High** | Implement 10-attempt bounded retry with 200ms exponential backoff; register `TaskbarCreated` (`KW-F-002`). | Phase 1, Step 5 (Attachment Spike) |
| **IR-002** | **Undocumented `0x052C` Spawn Message** | `[UNDOC_BEHAVIOR]` / `[OBSERVED_COMMUNITY]` | Windows 8, 8.1, 10 (all builds), Windows 11 (21H2, 22H2, 23H2) | **Fails on Windows 11 24H2 when desktop icons are hidden ("Show desktop icons" disabled)** | **CRITICAL** | **Dual-Mode Algorithm:** Check icon visibility. If icons hidden or WorkerW vanishes, attach directly beneath `Progman` via `SetWindowPos` (`KW-F-001`, `UDR-002`). | Phase 1, Step 5 (24H2 Attachment Spike) |
| **IR-003** | **WorkerW Z-Order Hierarchy Traversal** (`EnumWindows`) | `[DOC_API]` (API) / `[UNDOC_BEHAVIOR]` (Hierarchy) | Windows 10 & Windows 11 | Multiple `WorkerW` windows exist (DWM screen capture, multi-monitors, lockscreen); wrong window targeted | **High** | Search specifically for the `WorkerW` window whose immediate sibling in Z-order contains `SHELLDLL_DefView`. | Phase 1, Step 5 (Attachment Spike) |
| **IR-004** | **Sub-Desktop Window Parenting** (`SetParent(hPlayer, hWorkerW)`) | `[DOC_API]` | Windows NT 4.0 through Win 11 | Incompatible window styles cause player window to capture desktop mouse clicks or disappear | **High** | Apply strict Win32 styles: `WS_CHILD | WS_VISIBLE | WS_CLIPSIBLINGS`; remove `WS_POPUP` and `WS_CAPTION`. | Phase 1, Step 5 (Attachment Spike) |
| **IR-005** | **Desktop Icon Hit-Testing & Click-Through** | `[DOC_API]` | Windows 10 & Windows 11 | Player window intercepts input, preventing icon dragging, double-clicking, or marquee selection | **CRITICAL** | Window styles must ensure `hPlayer` sits strictly *behind* `SHELLDLL_DefView`; mouse clicks pass naturally to icons. | Phase 1, Step 5 (Hit-Test Verification) |
| **IR-006** | **Explorer Restart Recovery** (`RegisterWindowMessageW("TaskbarCreated")`) | `[DOC_API]` | All Windows NT releases | Rapid re-attachment attempts race with Explorer window initialization | **High** | Intercept `TaskbarCreated`; wait 500ms debounce period for shell hierarchy to stabilize before re-parenting. | Phase 1, Step 5 (Recovery Spike) |
| **IR-007** | **Mixed-DPI Coordinate Normalization** (`WM_DPICHANGED`) | `[DOC_API]` | Windows 10 Build 19041+ & Windows 11 | DWM virtualizes coordinates across 100% and 200% displays; player window scales blurrily | **Medium** | Declare `PerMonitorV2` DPI awareness in manifest; query `GetDpiForMonitor` and position using physical pixels (`KW-F-024`). | Phase 1, Step 6 (Multi-Monitor Spike) |
| **IR-008** | **Virtual Desktop Coordinate Offsets** (`GetSystemMetrics(SM_XVIRTUALSCREEN)`) | `[DOC_API]` | Windows 10 & Windows 11 | Secondary monitors positioned above/left of primary produce negative virtual coordinates | **Medium** | Normalize coordinates: $\Delta X = rcMonitor.left - SM\_XVIRTUALSCREEN$; compute viewport crop geometry (`KW-F-021`). | Phase 1, Step 6 (Multi-Monitor Spike) |
| **IR-009** | **Sleep / Modern Standby Wake Device Reset** (`DXGI_ERROR_DEVICE_RESET`) | `[DOC_API]` | Windows 10 & Windows 11 | Laptop sleep/wake invalidates Direct3D 11 swapchains, causing video players to crash | **High** | Intercept `WM_POWERBROADCAST`; catch DXGI errors, release device context, and rebuild swapchain from scratch within 2.0s (`KW-NF-008`). | Phase 1, Step 7 (Media Resilience Spike) |
| **IR-010** | **Fullscreen Detection False Positives** (`SHQueryUserNotificationState`) | `[DOC_API]` | Windows 10 & Windows 11 | Start menu, taskbar previews, or transparent overlays falsely trigger wallpaper pause | **Medium** | Filter out known shell window classes (`Shell_TrayWnd`, `Progman`, `WorkerW`, `Windows.UI.Core.CoreWindow`). | Phase 1, Step 8 (Foreground Hook Spike) |

---

## 3. The Windows 11 24H2 Composition Quirk & Dual-Mode Solution

### 3.1 Empirical Root Cause Analysis
In Windows 11 Version 24H2 (Build 26100+), Microsoft modified how DWM and `explorer.exe` maintain desktop surface composition:
- Under standard conditions with desktop icons visible, sending `0x052C` to `Progman` spawns a background `WorkerW` window sibling that hosts the wallpaper layer behind `SHELLDLL_DefView`.
- **The Failure State:** If the user hides desktop icons (via the desktop right-click menu or third-party tweakers), Explorer collapses the standard icon window hierarchy. In naive live wallpaper engines, sending `0x052C` spawns a transient `WorkerW` that DWM immediately closes or detaches upon the next render cycle, triggering **an infinite 60 Hz respawn loop, screen flashing, and grey screens** (`PSR-005`).

### 3.2 KinetiWall Dual-Mode Shell Attachment Algorithm
To guarantee stability across all Windows 11 24H2 environments, KinetiWall implements an adaptive, two-phase attachment routine:

```
[ Attachment Routine Invoked ]
              |
              v
[ Query Shell: FindWindowW("Progman") ]
              |
              v
[ Check Icon Visibility: FindWindowExW(hProgman, NULL, "SHELLDLL_DefView", NULL) ]
              |
       +------+------+
       |             |
(Icons Visible)  (Icons Hidden / DefView in WorkerW)
       |             |
       v             v
[ Path 1: WorkerW Injection ]      [ Path 2: Direct Progman Attachment ]
- Send 0x052C to Progman            - Skip WorkerW spawn
- Locate WorkerW sibling beneath     - SetParent(hPlayer, hProgman)
  SHELLDLL_DefView                 - SetWindowPos(hPlayer, HWND_BOTTOM, ...)
- SetParent(hPlayer, hWorkerW)       - Sits directly on desktop canvas
       |             |
       +------+------+
              |
              v
[ Verify Attachment Success via GetWindowRect & IsWindowVisible ]
              |
       +------+------+
       |             |
   (Success)      (Failed after 3 retries)
       |             |
       v             v
[ Render Live Wallpaper ]         [ Fallback: Restore Static Windows Wallpaper ]
                                  [ Surface Diagnostic Notification (KW-F-003) ]
```

---

## 4. Capability Detection & Fallback Principles

1. **No Hardcoded Build Branching:** The engine does **not** check whether `build == 26100` to decide how to parent windows. It checks the live window hierarchy state dynamically at runtime.
2. **Graceful Attachment Fallback (`KW-F-003`):** If both WorkerW injection and Progman parenting fail after 3 bounded retry attempts (with 250ms backoff), KinetiWall immediately restores the standard user desktop wallpaper via `SystemParametersInfoW(SPI_SETDESKWALLPAPER)` and logs a diagnostic message without crashing the application.
3. **Restoration on Termination (`KW-F-004`):** When KinetiWall exits, `kinetiwall-core` sends `WM_CLOSE` to all player windows, unparents child HWNDs, and calls `RedrawWindow` on `Progman` to ensure the desktop background redraws cleanly.

