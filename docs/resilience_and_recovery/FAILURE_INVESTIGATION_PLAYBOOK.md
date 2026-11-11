# FAILURE INVESTIGATION PLAYBOOK
## Document ID: KW-RCV-005 | Phase 3: Observability, Diagnostics & Operational Reliability

---

### 1. Operational Overview & Investigation Principles

This playbook provides actionable, step-by-step diagnostic workflows for software engineers, QA specialists, and technical support teams investigating failures in KinetiWall.

All investigations utilize the structured JSON Lines logs (`%LOCALAPPDATA%\KinetiWall\logs\`) defined in **Step 13 (`KW-OBS-001`)** and the local diagnostic export utility.

> [!CAUTION]
> **Privacy Invariant**: Support engineers MUST NEVER ask users for personal passwords, full disk paths, or unredacted system memory dumps. Instruct users to use the built-in *Export Diagnostics* feature, which guarantees automated sanitization.

---

### 2. Systematic Troubleshooting Workflows

#### 2.1 Workflow 1: Application Fails to Start (Startup Stall or Crash)
- **Symptoms**: KinetiWall tray icon does not appear; process launches and terminates immediately.
- **Diagnostic Procedure**:
  1. Open PowerShell and query the most recent Core log:
     ```powershell
     Get-Content "$env:LOCALAPPDATA\KinetiWall\logs\kinetiwall_core_*.log" -Tail 25 | ConvertFrom-Json | Select-Object timestamp_utc, severity, event_name, error_code, message
     ```
  2. Inspect the terminal event:
     - If `event_name == "APP_STARTUP_FAILED"` and `error_code == "ERR_PIPE_ALREADY_EXISTS"`: A secondary KinetiWall instance is already running in this logon session. Check Task Manager for existing `kinetiwall-core.exe`.
     - If `error_code == "ERR_CFG_PARSE_FAILED"`: `config.json` was corrupted. Check if `config.json.bak` exists. If not, inspect `config.json.corrupt_{timestamp}` for syntax errors.
     - If `error_code == "ERR_WIN_DESKTOP_HOOK_FAILED"`: Windows Explorer shell is not running or WorkerW handle could not be established. Ensure `explorer.exe` is running.

#### 2.2 Workflow 2: Wallpaper Playback Crashes Immediately or Enters Crash Loop
- **Symptoms**: Wallpaper flashes briefly on the desktop, disappears, and desktop Action Center displays: *"Wallpaper on Display 1 crashed repeatedly and has been stopped [KW-ERR-511]"*.
- **Diagnostic Procedure**:
  1. Query player crash events in the Core log:
     ```powershell
     Get-Content "$env:LOCALAPPDATA\KinetiWall\logs\kinetiwall_core_*.log" | Select-String "PLAYER_CRASH_DETECTED" | ConvertFrom-Json | Select-Object timestamp_utc, error_code, context
     ```
  2. Inspect the `error_code`:
     - If `STATUS_ACCESS_VIOLATION (0xC0000005)`: Native crash inside `mpv-2.dll` or GPU driver. Verify GPU driver version. Update Intel/NVIDIA/AMD drivers to latest WHQL build.
     - If `ERR_MED_UNSUPPORTED_CODEC`: The video profile (e.g. 10-bit HEVC or AV1) is not supported by the hardware decoder. Convert video to standard 8-bit H.264 MP4.
     - If `SUPERVISOR_CRASH_LOOP_TRIPPED`: The circuit breaker tripped after 3 rapid crashes. The wallpaper package has been quarantined. User must select a different wallpaper in the UI to reset the circuit breaker.

#### 2.3 Workflow 3: Fullscreen Pause Policy Not Triggering (Game Stutter)
- **Symptoms**: Video wallpaper continues rendering while a full-screen 3D game is active, causing frame drops.
- **Diagnostic Procedure**:
  1. Inspect foreground detection events:
     ```powershell
     Get-Content "$env:LOCALAPPDATA\KinetiWall\logs\kinetiwall_core_*.log" | Select-String "PAUSE_POLICY" | ConvertFrom-Json | Select-Object timestamp_utc, event_name, context
     ```
  2. Verify:
     - Did `PAUSE_POLICY_ACTIVATED` fire when the game launched?
     - If NOT: Check `context.foreground_class`. Some borderless windowed games execute in non-exclusive modes. Open KinetiWall *Settings $\to$ Performance* and ensure *"Pause on Maximized Windows"* is enabled in addition to *"Pause on Fullscreen Apps"*.

#### 2.4 Workflow 4: Unexpected High Background CPU Utilization ($> 2\%$)
- **Symptoms**: `kinetiwall-player-video.exe` consumes 8–15% CPU on a modern system.
- **Diagnostic Procedure**:
  1. Check if hardware acceleration is active:
     ```powershell
     Get-Content "$env:LOCALAPPDATA\KinetiWall\logs\player_video_*.log" | Select-String "RENDERER" | ConvertFrom-Json | Select-Object timestamp_utc, event_name, context
     ```
  2. If `event_name == "RENDERER_HW_ACCEL_FALLBACK"`:
     - Direct3D11VA hardware acceleration failed; player is using software CPU decoding via FFmpeg.
     - Cause: Outdated graphics driver, missing Direct3D11 feature level, or excessive video resolution ($> 4K$) for the installed GPU.

#### 2.5 Workflow 5: Multi-Monitor Display Glitch or Detachment Freeze
- **Symptoms**: Unplugging a secondary monitor causes desktop icons to flicker or wallpaper to stretch across displays incorrectly.
- **Diagnostic Procedure**:
  1. Query display topology events:
     ```powershell
     Get-Content "$env:LOCALAPPDATA\KinetiWall\logs\kinetiwall_core_*.log" | Select-String "DISPLAY_TOPOLOGY" | ConvertFrom-Json | Select-Object timestamp_utc, event_name, context
     ```
  2. Check if `MONITOR_DETACHMENT_DETECTED` fired.
  3. Verify that the Core suspended the player associated with the detached display (`PLAYER_TERMINATION_REQUESTED`) without touching the primary monitor's player.

---

### 3. Step-by-Step Diagnostic Package Export (User Guide)

When escalating an issue to the developer team:
1. Open KinetiWall Gallery UI from the system tray icon.
2. Click **Settings (Gear Icon)** $\to$ **About KinetiWall**.
3. Under *Diagnostics & Troubleshooting*, click **"Export Diagnostic Package"**.
4. A preview window will display the sanitized manifest (`manifest.json`) and scrubbed logs.
5. Click **"Save ZIP to Desktop"**.
6. Attach `KinetiWall_Diagnostics_{Date}.zip` to the support ticket.

