# Monitor Identity & Wallpaper Assignment Strategy
**Document ID:** `KW-DSP-002`  
**Deliverable:** B (Monitor Identity and Assignment Strategy)  
**Parent Document:** [README.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/display_management/README.md)  
**Classification:** System Architecture & Data Integrity Standard  

---

## 1. The Volatility of Windows Display Identifiers

A notorious bug in desktop personalization software is the swapping or loss of wallpaper assignments after system reboots, laptop docking/undocking, or GPU driver updates.

### 1.1 Why Win32 Device Names Fail as Persistent Keys
* **GDI Device Names (`\\.\DISPLAY1`, `\\.\DISPLAY2`):** Windows reassigns these names dynamically based on the order in which GPU display ports initialize during boot. If a DisplayPort monitor initializes 10 ms slower than an HDMI monitor, Windows assigns `\\.\DISPLAY1` to the HDMI monitor and `\\.\DISPLAY2` to the DisplayPort monitor, completely reversing previous assignments.
* **Enumeration Indices (`0`, `1`, `2`):** Ephemeral integers that change whenever any monitor is disconnected.
* **Monitor Handles (`HMONITOR`):** Opaque pointer-sized integer handles allocated by User32. Valid only for the duration of the current boot session; invalidated when displays reconnect.

---

## 2. Dual-Key Persistent Identity Strategy

To guarantee that user wallpaper assignments strictly track physical monitors across driver reloads, port changes, and dockings, KinetiWall implements a **Dual-Key Identity Resolution Algorithm**:

```
+---------------------------------------------------------------------------------+
|                      PRIMARY KEY: Hardware EDID SHA-256 Hash                    |
|  - Extracted from 128-byte Extended Display Identification Data (EDID)          |
|  - Components: Manufacturer ID (e.g. "DEL") + Product Code + Serial Number     |
|  - Algorithm: SHA-256(ManufacturerID || ProductCode || SerialNumber)           |
|  - Invariant: 100% persistent across port swaps, driver updates, and reboots!   |
+---------------------------------------------------------------------------------+
                                       |
                                       | Fallback if EDID unavailable / generic
                                       v
+---------------------------------------------------------------------------------+
|                     SECONDARY KEY: Windows Monitor Device Path                  |
|  - Device Instance ID from SetupAPI (e.g. "DISPLAY\DEL4170\5&2e91a0c&0&UID123") |
|  - Ties the assignment to the specific GPU port and hardware device instance.   |
+---------------------------------------------------------------------------------+
                                       |
                                       | Fallback if multi-monitor cloning
                                       v
+---------------------------------------------------------------------------------+
|                     TERTIARY FALLBACK: GDI Device Name                          |
|  - Fallback key: "\\.\DISPLAY1"                                                 |
|  - Used only for virtual displays (e.g. Remote Desktop, VirtualBox, IDD)       |
+---------------------------------------------------------------------------------+
```

### 2.1 Compatibility with Step 9 Configuration Schema

This strategy strictly complies with the approved configuration schema established in `docs/data_architecture/CONFIGURATION_AND_PROPERTY_SCHEMAS.md`:

```json
{
  "display_profiles": {
    "monitors": {
      "edid:e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855": {
        "hardware_edid_hash": "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855",
        "display_name": "DELL U2720Q (DisplayPort-1)",
        "assigned_wallpaper_id": "a4d3f2b1-6c5e-4a9b-8e7d-1c3b5a7e9f2d",
        "stretch_mode": "fill"
      }
    }
  }
}
```

---

## 3. Assignment Resolution & Matching Algorithm

When KinetiWall initializes or receives a `WM_DISPLAYCHANGE` notification, it executes the following resolution algorithm for every detected physical monitor:

```rust
pub fn match_display_to_assignment(
    connected_monitor: &DisplayDescriptor,
    saved_profile: &DisplayProfileConfig,
) -> AssignmentMatchResult {
    // Stage 1: Exact EDID Hash Match
    if let Some(assignment) = saved_profile.monitors.get(&connected_monitor.hardware_edid_hash) {
        return AssignmentMatchResult::Matched(assignment.clone());
    }

    // Stage 2: Secondary Device Path Match
    for (key, assignment) in &saved_profile.monitors {
        if assignment.hardware_edid_hash.is_empty() && assignment.display_name == connected_monitor.friendly_name {
            return AssignmentMatchResult::Matched(assignment.clone());
        }
    }

    // Stage 3: Tertiary GDI Device Name Match (Virtual Displays)
    if let Some(assignment) = saved_profile.monitors.get(&connected_monitor.gdi_device_name) {
        return AssignmentMatchResult::Matched(assignment.clone());
    }

    // Stage 4: Unassigned New Display
    AssignmentMatchResult::Unassigned
}
```

### 3.1 Handling Disconnected or Missing Monitors
* **Rule:** A disconnected display's assignment is **never deleted or overwritten** in `config.json`.
* **State Behavior:** The supervisor marks the corresponding monitor slot `STATUS_DISCONNECTED`. Active player processes for that display are terminated within $250\text{ ms}$, freeing GPU/VRAM resources. When the user reconnects the monitor (even days later or to a different display port), KinetiWall matches its EDID hash and instantly restores its wallpaper.

### 3.2 Handling Ambiguous Monitors (Identical Dual Monitors)
* **Hazard:** A user connects two identical monitors of the same brand and model (e.g. two Dell U2720Q displays). In budget displays, manufacturers sometimes program identical serial numbers in the EDID block (`00000000`).
* **Mitigation:** If two connected monitors report identical EDID hashes, KinetiWall disambiguates them using their **GPU Video Output Port ID** (obtained via `EnumDisplayDevicesW` adapter device paths). The primary key becomes:
  $$\text{DisambiguatedKey} = \text{SHA-256}(\text{EDID} \mathbin{\Vert} \text{GPU\_Port\_Instance})$$

---

## 4. Privacy & Telemetry Redaction

Hardware EDID blocks contain raw manufacturer serial numbers. To comply with Step 10 and Step 13 privacy standards:
1. **Raw Serial Numbers are Never Persisted:** Only the one-way cryptographic SHA-256 hash (`hardware_edid_hash`) is stored in `config.json` or emitted in logs.
2. **Telemetry Redaction:** Diagnostics and bug report dumps scrub exact EDID serial numbers, displaying only the friendly model name (e.g. `LG UltraGear 27GN950`) and the truncated hash (`edid:e3b0c4...`).

