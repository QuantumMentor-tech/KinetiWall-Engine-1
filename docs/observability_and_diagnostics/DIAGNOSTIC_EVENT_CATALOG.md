# DIAGNOSTIC EVENT CATALOG
## Document ID: KW-OBS-005 | Phase 3: Observability, Diagnostics & Operational Reliability

---

### 1. Catalog Overview & Event Naming Taxonomy

This catalog defines the stable, standardized diagnostic events emitted by KinetiWall across its multi-process lifecycle. Event names follow the `DOMAIN_ACTION_STATUS` taxonomy in screaming snake case.

```
+---------------------------------------------------------------------------------------------------------+
|                                  DIAGNOSTIC EVENT DOMAINS (50+ EVENTS)                                  |
+--------+------------------------------------+-----------+-----------------------------------------------+
| DOMAIN | DOMAIN NAME                        | ID PREFIX | SCOPE & RESPONSIBILITIES                      |
+--------+------------------------------------+-----------+-----------------------------------------------+
| 01     | Application Lifecycle              | APP_xxx   | Engine startup, UI lifecycle, clean shutdown  |
| 02     | Process Lifecycle                  | PROC_xxx  | Player spawn, watchdogs, crash recovery       |
| 03     | Wallpaper Lifecycle                | WALL_xxx  | Selection, validation, play, pause, resume    |
| 04     | Desktop & Display Integration      | DSK_xxx   | WorkerW reparenting, fullscreen detection     |
| 05     | IPC & Coordination                 | IPC_xxx   | Named Pipe connection, framing, timeouts      |
| 06     | Runtime & Renderer Capabilities    | RNT_xxx   | D3D11 swapchains, mpv, WebView2 sandboxing    |
| 07     | Storage & Configuration            | STG_xxx   | Config persistence, log rotation, exports     |
| 08     | Performance & System Health        | HLT_xxx   | Frame drop alerts, queue overflow, watchdogs  |
+--------+------------------------------------+-----------+-----------------------------------------------+
```

---

### 2. Domain 01: Application Lifecycle Events

| Event ID | Event Name | Component | Severity | Mandatory Context Fields | Sensitive-Content Restrictions | Rate Class |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| `APP_001` | `APP_STARTUP_INITIATED` | `core` | `INFO` | `engine_version`, `windows_build` | No usernames or profile paths. | Infrequent |
| `APP_002` | `APP_CONFIG_LOADED` | `core` | `INFO` | `schema_version`, `monitor_count` | Zero custom text strings. | Infrequent |
| `APP_003` | `APP_STARTUP_COMPLETED` | `core` | `INFO` | `duration_ms`, `autostart` | No personal identifiers. | Infrequent |
| `APP_004` | `APP_UI_OPENED` | `ui` | `INFO` | `ui_version`, `display_mode` | No active document names. | On-demand |
| `APP_005` | `APP_UI_CLOSED` | `ui` | `INFO` | `duration_open_sec` | None. | On-demand |
| `APP_006` | `APP_SHUTDOWN_INITIATED`| `core` | `INFO` | `trigger_reason` | None. | Infrequent |
| `APP_007` | `APP_SHUTDOWN_COMPLETED`| `core` | `INFO` | `total_uptime_sec` | None. | Infrequent |
| `APP_008` | `APP_STARTUP_FAILED` | `core` | `FATAL` | `error_code`, `failure_stage` | Mask all path segments. | Exceptional|
| `APP_009` | `APP_PANIC_ENCOUNTERED` | `*` | `FATAL` | `error_code`, `location`, `thread`| Redact user paths in location. | Exceptional|

---

### 3. Domain 02: Process Lifecycle Events

| Event ID | Event Name | Component | Severity | Mandatory Context Fields | Sensitive-Content Restrictions | Rate Class |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| `PROC_001`| `PLAYER_SPAWN_REQUESTED` | `core` | `INFO` | `target_monitor_id`, `player_type`| No file paths; UUID only. | On-demand |
| `PROC_002`| `PLAYER_PROCESS_STARTED` | `core` | `INFO` | `player_pid`, `player_instance_id`| None. | On-demand |
| `PROC_003`| `PLAYER_STARTUP_TIMED_OUT`| `core` | `ERROR` | `target_monitor_id`, `timeout_ms`| None. | Exceptional|
| `PROC_004`| `PLAYER_PROCESS_EXIT_NORMAL`| `core` | `INFO` | `player_pid`, `exit_code: 0` | None. | On-demand |
| `PROC_005`| `PLAYER_CRASH_DETECTED` | `core` | `ERROR` | `crashed_pid`, `exit_code`, `attempt`| No stack traces in context. | Exceptional|
| `PROC_006`| `PLAYER_TERMINATION_REQUESTED`| `core`| `INFO`| `target_pid`, `reason` | None. | On-demand |
| `PROC_007`| `PLAYER_RESOURCES_RELEASED`| `core`| `INFO` | `target_monitor_id` | None. | On-demand |
| `PROC_008`| `RECOVERY_CYCLE_STARTED` | `core` | `WARN` | `target_monitor_id`, `backoff_ms`| None. | Exceptional|
| `PROC_009`| `RECOVERY_CYCLE_COMPLETED`| `core` | `INFO` | `target_monitor_id`, `new_pid` | None. | Exceptional|
| `PROC_010`| `SUPERVISOR_CRASH_LOOP_TRIPPED`| `core`| `FATAL`| `crashes_in_window`, `quarantine`| No user text strings. | Exceptional|

---

### 4. Domain 03: Wallpaper Lifecycle Events

| Event ID | Event Name | Component | Severity | Mandatory Context Fields | Sensitive-Content Restrictions | Rate Class |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| `WALL_001`| `WALLPAPER_SELECTION_REQUESTED`| `core`| `INFO`| `wallpaper_id`, `monitor_id` | Package UUID only; no paths. | On-demand |
| `WALL_002`| `WALLPAPER_PACKAGE_VALIDATED` | `core`| `INFO`| `wallpaper_id`, `package_type`| No raw author personal info. | On-demand |
| `WALL_003`| `WALLPAPER_PLAYER_INITIALIZED`| `player`| `INFO`| `wallpaper_id`, `duration_ms` | None. | On-demand |
| `WALL_004`| `WALLPAPER_FIRST_FRAME_PRESENTED`| `player`| `INFO`| `duration_ms`, `resolution` | None. | On-demand |
| `WALL_005`| `WALLPAPER_PLAYBACK_ACTIVE` | `player`| `INFO`| `fps`, `codec`, `volume` | None. | On-demand |
| `WALL_006`| `WALLPAPER_PAUSED` | `player`| `INFO` | `trigger_reason`, `latency_ms`| No foreground window titles. | Frequent |
| `WALL_007`| `WALLPAPER_RESUMED` | `player`| `INFO` | `latency_ms`, `resumed_frame` | None. | Frequent |
| `WALL_008`| `WALLPAPER_STOPPED` | `player`| `INFO` | `stop_reason` | None. | On-demand |
| `WALL_009`| `WALLPAPER_SWITCH_COMPLETED`| `core` | `INFO` | `previous_id`, `new_id`, `duration`| UUIDs only. | On-demand |
| `WALL_010`| `WALLPAPER_SWITCH_FAILED` | `core` | `ERROR` | `error_code`, `attempted_id` | Mask error path strings. | Exceptional|

---

### 5. Domain 04: Desktop & Display Integration Events

| Event ID | Event Name | Component | Severity | Mandatory Context Fields | Sensitive-Content Restrictions | Rate Class |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| `DSK_001` | `DESKTOP_INTEGRATION_INITIALIZED`| `core`| `INFO`| `target_hwnd_type: "WorkerW"`| None. | Infrequent |
| `DSK_002` | `DESKTOP_INTEGRATION_FAILED` | `core`| `ERROR`| `error_code`, `win32_last_error`| None. | Exceptional|
| `DSK_003` | `DISPLAY_TOPOLOGY_CHANGED` | `core`| `INFO` | `old_count`, `new_count` | No serial numbers in context.| Infrequent |
| `DSK_004` | `MONITOR_ATTACHMENT_DETECTED` | `core`| `INFO` | `monitor_id`, `resolution` | Hash EDID serials. | Infrequent |
| `DSK_005` | `MONITOR_DETACHMENT_DETECTED` | `core`| `INFO` | `monitor_id` | None. | Infrequent |
| `DSK_006` | `FOREGROUND_WINDOW_CHANGED` | `core`| `DEBUG`| `window_state`, `process_name` | **STRICT: NO WINDOW TITLES**. | Frequent |
| `DSK_007` | `PAUSE_POLICY_ACTIVATED` | `core`| `INFO` | `policy: "fullscreen"`, `monitor`| Obfuscate process details. | Frequent |
| `DSK_008` | `PAUSE_POLICY_DEACTIVATED` | `core`| `INFO` | `policy: "fullscreen"`, `monitor`| None. | Frequent |
| `DSK_009` | `DESKTOP_RESTORATION_COMPLETED`| `core`| `INFO`| `detached_monitors_count` | None. | Infrequent |

---

### 6. Domain 05: IPC & Coordination Events

| Event ID | Event Name | Component | Severity | Mandatory Context Fields | Sensitive-Content Restrictions | Rate Class |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| `IPC_001` | `IPC_ENDPOINT_LISTENING` | `core` | `INFO` | `pipe_name_pattern` | Session ID only; no secrets. | Infrequent |
| `IPC_002` | `IPC_CLIENT_CONNECTED` | `core` | `DEBUG`| `client_pid`, `endpoint_id` | None. | On-demand |
| `IPC_003` | `IPC_CLIENT_AUTHENTICATED` | `core` | `INFO` | `client_pid`, `role: "player"` | None. | On-demand |
| `IPC_004` | `IPC_REQUEST_TIMED_OUT` | `*` | `WARN` | `payload_name`, `timeout_ms` | No payload contents. | Exceptional|
| `IPC_005` | `IPC_MESSAGE_MALFORMED` | `core` | `ERROR` | `error_code`, `bytes_read` | Do not log raw packet body. | Exceptional|
| `IPC_006` | `IPC_CLIENT_DISCONNECTED` | `core` | `INFO` | `client_pid`, `normal_exit` | None. | On-demand |
| `IPC_007` | `IPC_PROTOCOL_MISMATCH` | `core` | `ERROR` | `client_version`, `expected: 1`| None. | Exceptional|

---

### 7. Domain 06: Runtime & Renderer Capabilities Events

| Event ID | Event Name | Component | Severity | Mandatory Context Fields | Sensitive-Content Restrictions | Rate Class |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| `RNT_001` | `RENDERER_D3D11_INITIALIZED` | `player`| `INFO` | `feature_level`, `adapter_name`| None. | On-demand |
| `RNT_002` | `RENDERER_HW_ACCEL_FALLBACK` | `player`| `WARN` | `codec`, `fallback: "software"`| None. | Exceptional|
| `RNT_003` | `WEBVIEW2_RUNTIME_DETECTED` | `core` | `INFO` | `version`, `is_evergreen` | None. | Infrequent |
| `RNT_004` | `WEBVIEW2_SANDBOX_INITIALIZED`| `player`| `INFO`| `virtual_host: "kinetiwall.local"`| No external URLs. | On-demand |
| `RNT_005` | `WASAPI_LOOPBACK_INITIALIZED` | `core` | `INFO` | `sample_rate`, `channels` | **ZERO RAW PCM / AUDIO DATA**.| On-demand |
| `RNT_006` | `WASAPI_STREAM_STOPPED` | `core` | `INFO` | `duration_sec` | None. | On-demand |

---

### 8. Domain 07: Storage, Configuration & Diagnostics Events

| Event ID | Event Name | Component | Severity | Mandatory Context Fields | Sensitive-Content Restrictions | Rate Class |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| `STG_001` | `CONFIG_WRITE_COMPLETED` | `core` | `INFO` | `schema_version`, `duration_ms`| Zero custom string values. | Infrequent |
| `STG_002` | `CONFIG_WRITE_FAILED` | `core` | `ERROR` | `error_code`, `win32_code` | Mask file paths. | Exceptional|
| `STG_003` | `LOG_ROTATION_COMPLETED` | `*` | `INFO` | `archived_file`, `new_file` | Mask username path segments. | Daily |
| `STG_004` | `LOG_RETENTION_PRUNING_COMPLETED`| `core`| `INFO` | `pruned_files_count`, `freed_bytes`| None. | Daily |
| `STG_005` | `DIAG_EXPORT_COMPLETED` | `core` | `INFO` | `output_zip_size_bytes` | Mask destination folder path.| On-demand |
| `STG_006` | `DIAG_DISK_FULL_FALLBACK` | `*` | `WARN` | `in_memory_buffer_capacity` | None. | Exceptional|

---

### 9. Domain 08: Performance & System Health Events

| Event ID | Event Name | Component | Severity | Mandatory Context Fields | Sensitive-Content Restrictions | Rate Class |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| `HLT_001` | `HEALTH_CHECK_PASSED` | `core` | `DEBUG`| `active_players_count`, `ram_mb`| None. | Periodic |
| `HLT_002` | `HEALTH_CHECK_WARNING` | `core` | `WARN` | `unresponsive_pid`, `heartbeat`| None. | Exceptional|
| `HLT_003` | `PERF_BUDGET_EXCEEDED` | `core` | `WARN` | `metric_code`, `observed`, `budget`| None. | Exceptional|
| `HLT_004` | `PRESENTATION_FRAME_DROPPED` | `player`| `WARN` | `dropped_count`, `target_fps` | None. | Exceptional|
| `HLT_005` | `DIAG_LOG_QUEUE_OVERFLOW` | `*` | `WARN` | `dropped_events_count` | None. | Exceptional|

