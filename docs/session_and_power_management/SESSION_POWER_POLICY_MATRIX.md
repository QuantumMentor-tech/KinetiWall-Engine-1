# Windows Session & Power Policy Matrix
**Document ID:** `KW-LCE-003`  
**Deliverable:** C (Session and Power Policy Matrix)  
**Parent Document:** [README.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/session_and_power_management/README.md)  
**Classification:** Operational Policy Reference / System Reliability Standard  

---

## 1. Master Session & Power Policy Matrix

The following authoritative matrix specifies KinetiWall's exact architectural response across all 15 recognized Windows session, power, and lifecycle transitions:

| Event Identifier | Native Ingress Signal | Policy Owner | Pause Reason Added / Removed | Player Subprocess Action | Desktop / Window State | Max Budget | Diagnostic Event |
| :--- | :--- | :--- | :--- | :--- | :--- | :---: | :--- |
| **`EVT-01: System Sleep Pending`** | `PBT_APMSUSPEND` | Core Supervisor | `+ SystemSleep` | Freeze mpv/WebView2; stop WASAPI stream | Kept attached to `WorkerW` | $150\text{ ms}$ | `PWR_SYSTEM_SLEEP_ENTER` |
| **`EVT-02: System Wake Automatic`**| `PBT_APMRESUMEAUTOMATIC`| Core Supervisor | `- SystemSleep` | Re-audit D3D11 device; unfreeze if mask empty | Re-verify `WorkerW` HWND | $250\text{ ms}$ | `PWR_SYSTEM_RESUME_AUTO` |
| **`EVT-03: System Wake User`** | `PBT_APMRESUMESUSPEND` | Core Supervisor | `- SystemSleep` | Re-present last frame; resume 60 FPS playback | Ensure z-order beneath icons | $100\text{ ms}$ | `PWR_SYSTEM_RESUME_USER` |
| **`EVT-04: Display Power Off`** | `GUID_MONITOR_POWER_ON` (0)| Core Supervisor | `+ DisplayPowerOff` | Freeze render loop; halt DXGI `Present` | Retain `WorkerW` child bounds | $20\text{ ms}$ | `PWR_DISPLAY_POWER_OFF` |
| **`EVT-05: Display Power On`** | `GUID_MONITOR_POWER_ON` (1)| Core Supervisor | `- DisplayPowerOff` | Resume swapchain presentation | Re-verify physical geometry | $50\text{ ms}$ | `PWR_DISPLAY_POWER_ON` |
| **`EVT-06: Workstation Lock`** | `WTS_SESSION_LOCK` | Core Supervisor | `+ SessionLocked` | Pause video decoding; stop audio capture | Retain `WorkerW` child bounds | $20\text{ ms}$ | `WTS_LOCK_ENTER` |
| **`EVT-07: Workstation Unlock`** | `WTS_SESSION_UNLOCK` | Core Supervisor | `- SessionLocked` | Resume playback if mask empty | Re-verify shell icon layers | $30\text{ ms}$ | `WTS_UNLOCK_RESUME` |
| **`EVT-08: Battery Power Switch`** | `GUID_ACDC_POWER_SOURCE` (1)| Core Supervisor | `+ BatterySaving` (if opt)| Apply battery throttle or pause per settings | Retain `WorkerW` child bounds | $50\text{ ms}$ | `PWR_SOURCE_BATTERY` |
| **`EVT-09: AC Power Switch`** | `GUID_ACDC_POWER_SOURCE` (0)| Core Supervisor | `- BatterySaving` | Restore full 60 FPS playback rate | Retain `WorkerW` child bounds | $50\text{ ms}$ | `PWR_SOURCE_AC` |
| **`EVT-10: Low Battery Critical`** | `GUID_BATTERY_PERCENTAGE` | Core Supervisor | `+ BatterySaving` | Force-pause video/web wallpaper playback | Restore static shell wallpaper| $50\text{ ms}$ | `PWR_BATTERY_CRITICAL` |
| **`EVT-11: User Sign-Out / Logoff`**| `WM_QUERYENDSESSION` | Core Supervisor | `+ EngineShutdown` | Clean teardown; release swapchains and exit | Detach & unparent HWND | $200\text{ ms}$ | `SYS_LOGOFF_TEARDOWN` |
| **`EVT-12: System Shutdown`** | `WM_ENDSESSION` (TRUE) | Core Supervisor | `+ EngineShutdown` | Terminate all players; close Job Object; exit| Expose static desktop | $250\text{ ms}$ | `SYS_SHUTDOWN_EXIT` |
| **`EVT-13: Fast User Switching`** | `WTS_CONSOLE_DISCONNECT` | Core Supervisor | `+ SessionDisconnected` | Pause rendering; mute audio loopbacks | Retain in background | $50\text{ ms}$ | `WTS_USER_SWITCH_OUT` |
| **`EVT-14: User Session Reconnect`**| `WTS_CONSOLE_CONNECT` | Core Supervisor | `- SessionDisconnected` | Re-audit active session ID; resume playback | Re-verify WorkerW host | $100\text{ ms}$ | `WTS_USER_SWITCH_IN` |
| **`EVT-15: Explorer Crash`** | `TaskbarCreated` | Core Supervisor | `+ HostReconciliation` | Detach HWND; discover new WorkerW; re-anchor | Re-parent into new WorkerW | $500\text{ ms}$ | `SHELL_EXPLORER_RESTART` |

---

## 2. Telemetry Serialization Schema

All lifecycle transitions generate structured, privacy-preserving log entries conforming to Step 13's diagnostic standard:

```json
{
  "timestamp": "2026-11-11T11:46:20.104Z",
  "level": "INFO",
  "component": "kinetiwall-core",
  "subsystem": "power_manager",
  "event_type": "LIFECYCLE_TRANSITION",
  "payload": {
    "event_id": "EVT-04",
    "event_name": "PWR_DISPLAY_POWER_OFF",
    "native_signal": "GUID_MONITOR_POWER_ON",
    "new_state": "STATE_DISPLAY_INACTIVE",
    "active_pause_reasons": ["DisplayPowerOff"],
    "affected_displays": ["\\\\.\\DISPLAY1", "\\\\.\\DISPLAY2"],
    "action": "HALT_SWAPCHAINS",
    "latency_ms": 14
  }
}
```

