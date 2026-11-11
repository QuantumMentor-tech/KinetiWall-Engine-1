# Lifecycle Policy Precedence & State Reference
**Document ID:** `KW-LCE-004`  
**Deliverable:** D (Lifecycle Policy Precedence and State Reference)  
**Parent Document:** [README.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/session_and_power_management/README.md)  
**Classification:** Operational Policy Reference / Architecture Standard  

---

## 1. The Multi-Reason Pause Concurrency Problem

In desktop wallpaper systems, pause conditions frequently overlap. For example:
1. The user explicitly pauses wallpaper playback via the system tray icon (`UserManual`).
2. While paused, the user launches a fullscreen game (`FullscreenOccluded`).
3. During gaming, the battery drops below 20% (`BatterySaving`).
4. The user locks the computer and walks away (`SessionLocked`).
5. When the user returns and unlocks the machine, the `SessionLocked` condition is removed.

A naive boolean implementation (`is_paused = false`) would mistakenly resume wallpaper playback immediately upon unlock, violating both the user's explicit manual pause and the battery-saving policy!

---

## 2. Composable Pause Reason Bitmask Architecture

KinetiWall solves this problem permanently by modeling pause state as a **Composable Reason Bitmask** (`PauseReasonSet`):

```rust
bitflags::bitflags! {
    /// Distinct, non-interfering reasons why a display's wallpaper is paused
    #[derive(Debug, Clone, Copy, PartialEq, Eq)]
    pub struct PauseReason: u32 {
        /// User explicitly clicked "Pause" via Tray or UI (Highest Precedence)
        const UserManual           = 1 << 0;
        /// System is in sleep / standby / hibernation
        const SystemSleep          = 1 << 1;
        /// Display monitor is powered off (GUID_MONITOR_POWER_ON = 0)
        const DisplayPowerOff      = 1 << 2;
        /// Workstation is locked (WTS_SESSION_LOCK)
        const SessionLocked        = 1 << 3;
        /// User switched to another Windows account / disconnected session
        const SessionDisconnected  = 1 << 4;
        /// A fullscreen foreground application covers the target display
        const FullscreenOccluded   = 1 << 5;
        /// Host running on battery below user threshold (e.g. < 20%)
        const BatterySaving        = 1 << 6;
        /// Engine is actively shutting down or logoff in progress
        const EngineShutdown       = 1 << 7;
        /// Explorer restart or display change reconciliation in progress
        const HostReconciliation   = 1 << 8;
    }
}
```

### 2.1 The Playback Execution Invariant

```rust
impl DisplayPlaybackController {
    /// Determines whether the player should actively render frames
    #[inline(always)]
    pub fn is_playback_permitted(&self) -> bool {
        // INVIOLABLE RULE: Playback is permitted IF AND ONLY IF the pause mask is EMPTY!
        self.active_pause_reasons.is_empty()
    }
}
```

* **Clearing a Condition:** When an event clears (e.g. `WTS_SESSION_UNLOCK`), only the `SessionLocked` bit is cleared from the mask. If `UserManual` or `BatterySaving` remains set, `is_playback_permitted()` returns `false`, and the player remains safely quiescent.
* **Resuming Playback:** Playback resumes **if and only if** the bitmask equals `0` (`PauseReason::empty()`).

---

## 3. Strict Precedence Hierarchy

When multiple lifecycle transitions occur simultaneously, operations are executed in order of strict architectural precedence:

```
[PRECEDENCE 1: TERMINAL / ENGINE SHUTDOWN]
  - WM_QUERYENDSESSION / WM_ENDSESSION / User clicks "Exit"
  - Cancels all in-flight reconciliations; initiates immediate teardown.
                    |
                    v
[PRECEDENCE 2: EXPLICIT USER MANUAL INTENT]
  - User clicks "Pause" or "Stop" via Tray or Settings GUI.
  - Immune to all OS wake, unlock, or battery AC transitions.
                    |
                    v
[PRECEDENCE 3: SYSTEM SLEEP & DISPLAY POWER OFF]
  - PBT_APMSUSPEND / GUID_MONITOR_POWER_ON = 0
  - Overrides active fullscreen or battery throttling.
                    |
                    v
[PRECEDENCE 4: SESSION LOCK & FAST USER SWITCH]
  - WTS_SESSION_LOCK / WTS_CONSOLE_DISCONNECT
  - Halts swapchains and mutes audio sessions.
                    |
                    v
[PRECEDENCE 5: FULLSCREEN HARDWARE OCCLUSION]
  - Active game or fullscreen video covers display.
  - Pauses local swapchain; resumes automatically when window loses focus.
                    |
                    v
[PRECEDENCE 6: BATTERY CONSERVATION]
  - Laptop running on battery below threshold (KW-F-008).
                    |
                    v
[PRECEDENCE 7: NORMAL ACTIVE PLAYBACK (60 FPS)]
  - Evaluated only when Precedence Levels 1 through 6 are clear.
```

---

## 4. Battery-Aware Threshold Policy (`KW-F-008`)

To preserve battery life on Windows laptops and portable tablets:
1. **Power Source Observer:** Core subscribes to `GUID_ACDC_POWER_SOURCE`. When switching from AC to Battery:
   - If user setting `pause_on_battery == true`, Core adds `PauseReason::BatterySaving`.
   - If `pause_on_battery == false`, playback continues, but frame rate is optionally throttled to 30 FPS to reduce GPU power draw.
2. **Critical Battery Threshold:** Core subscribes to `GUID_BATTERY_PERCENTAGE_REMAINING`. If battery drops below $15\%$, `PauseReason::BatterySaving` is added unconditionally, freezing live wallpapers and restoring the static Windows desktop wallpaper until AC power is restored.

