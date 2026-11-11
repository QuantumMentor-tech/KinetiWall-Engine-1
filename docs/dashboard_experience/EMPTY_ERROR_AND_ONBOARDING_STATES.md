# Dashboard Empty, Error & Onboarding States
**Document ID:** `KW-DSH-003`  
**Deliverable:** C (Empty, Error, and Onboarding States)  
**Parent Document:** [README.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/dashboard_experience/README.md)  
**Classification:** Frontend UX & Error Presentation Standard  

---

## 1. Distinct State Classification

A common UX defect in dashboard design is collapsing distinct operational conditions into a single generic error card. KinetiWall distinguishes between five explicit states:

```
+---------------------------------------------------------------------------------+
| DASHBOARD STATE TAXONOMY                                                        |
+---------------------------------------------------------------------------------+
| 1. OPERATIONAL ACTIVE:   Valid active wallpaper rendering at 60 FPS             |
| 2. UNASSIGNED ONBOARDING:Fresh install or library empty; no active wallpaper    |
| 3. ENGINE DISCONNECTED:  Core daemon initializing or IPC named pipe offline    |
| 4. HARDWARE DEGRADED:    Graphics device reset or WorkerW host invalidation     |
| 5. ASSET UNAVAILABLE:    Assigned wallpaper file missing or corrupt on disk     |
+---------------------------------------------------------------------------------+
```

---

## 2. State Blueprints & Visual Guidance

### 2.1 State 2: Unassigned Onboarding State ("No Active Wallpaper")

Rendered when KinetiWall is freshly installed or the user has cleared all wallpaper assignments:

```
+---------------------------------------------------------------------------------+
| [Hero Panel: Acrylic Glass Border]                                              |
|                                                                                 |
|   ✨ Welcome to KinetiWall Engine                                                |
|   No active live wallpaper is currently assigned to your desktop.               |
|                                                                                 |
|   Transform your desktop with hardware-accelerated 4K video, interactive        |
|   WebGL shaders, and animated desktop experiences.                              |
|                                                                                 |
|   [ 📁 Browse Wallpaper Library ]          [ ⚙️ Configure Displays ]            |
+---------------------------------------------------------------------------------+
```

* **Clear Call-to-Action:** The primary action button ("Browse Wallpaper Library") routes directly to the Library view, focusing the search input automatically.
* **No Artificial Data:** Omit frame rate, VRAM gauges, and play/pause controls; never show fake operational stats.

---

### 2.2 State 3: Engine Disconnected Banner

Rendered if `kinetiwall-core.exe` is still starting up or recovering from an abnormal crash:

```
+---------------------------------------------------------------------------------+
| ⚠️ Connecting to KinetiWall Core Supervisor...                                  |
| The background rendering daemon is initializing. Please wait.                   |
| [ Retrying connection in 3s... ]                              [ Diagnostics ]   |
+---------------------------------------------------------------------------------+
```

* **Non-Blocking Ingress:** Quick action buttons on the Dashboard are disabled with an informative tooltip until the IPC connection is confirmed.

---

### 2.3 State 4: Hardware Degraded Alert Banner

Rendered if graphics hardware experienced a device reset (`DXGI_ERROR_DEVICE_RESET`) or Windows Explorer crashed:

```
+---------------------------------------------------------------------------------+
| ⚠️ Desktop Graphics Recovery in Progress                                         |
| Windows graphics driver reset or desktop shell restart detected.                |
| KinetiWall is re-anchoring wallpapers to the Windows desktop surface.           |
| [ Re-anchor Now ]                                             [ View Logs ]     |
+---------------------------------------------------------------------------------+
```

* **Safe Static Fallback:** While degraded, the native Windows static desktop wallpaper is cleanly exposed, preventing black screens or frozen visual artifacts.

---

### 2.4 State 5: Missing / Corrupt Wallpaper Asset Card

Rendered if the user moved, deleted, or unmounted the disk drive containing the active wallpaper video file:

```
+---------------------------------------------------------------------------------+
| ❌ Active Wallpaper File Missing                                                |
| Could not locate video file for 'Cyberpunk City 4K'.                            |
| The file may have been moved, renamed, or deleted from your local drive.        |
| [ Select New Wallpaper ]                          [ Remove Assignment ]         |
+---------------------------------------------------------------------------------+
```

* **No Crash / Zero Panic:** The missing file is caught cleanly at the player service boundary; Core marks the slot faulted and the Dashboard guides the user to reassign a valid wallpaper.

