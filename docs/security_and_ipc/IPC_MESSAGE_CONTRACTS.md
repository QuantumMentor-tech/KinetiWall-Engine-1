# IPC MESSAGE CONTRACT CATALOG
## Document ID: KW-SEC-002 | Phase 2: System Architecture & Technical Decisions

---

### 1. Message Framing and Envelope Specification

To guarantee reliable boundary detection and prevent message fragmentation over Windows Named Pipe byte-streams, all KinetiWall IPC messages utilize **Length-Prefixed Framing**:

```
+-----------------------------------+-------------------------------------------------------------+
| 4-Byte Big-Endian Length (u32)    | JSON-Encoded Message Envelope Payload (Variable Length)     |
| [ 0x00, 0x00, 0x01, 0x4A ]        | { "protocol_version": 1, "message_type": "Command", ... }   |
+-----------------------------------+-------------------------------------------------------------+
```

#### 1.1 The Standard Message Envelope

Every transmitted message MUST conform to the root envelope structure:

```rust
use serde::{Deserialize, Serialize};
use uuid::Uuid;

#[derive(Debug, Clone, Serialize, Deserialize, PartialEq)]
pub struct IpcEnvelope {
    pub protocol_version: u32,               // Must equal 1 for V1.0
    pub message_id: Uuid,                   // Unique message identifier
    pub correlation_id: Option<Uuid>,       // Set on responses to link with requests
    pub timestamp_utc_ms: i64,             // UTC timestamp in milliseconds
    pub message_type: MessageType,          // Command | Event | Response
    pub payload_name: String,               // Identifier e.g. "REQ_SET_CONFIG"
    pub payload: serde_json::Value,         // Strictly typed inner payload
    pub error: Option<IpcErrorPayload>,      // Populated only on failure responses
}

#[derive(Debug, Clone, Copy, Serialize, Deserialize, PartialEq)]
#[serde(rename_all = "snake_case")]
pub enum MessageType {
    Command,   // Imperative instruction requiring execution & response
    Query,     // Read-only request requiring data response
    Response,  // Direct response to a Command or Query
    Event,     // One-way broadcast notification (no direct response expected)
}

#[derive(Debug, Clone, Serialize, Deserialize, PartialEq)]
pub struct IpcErrorPayload {
    pub code: String,       // e.g. "ERR_UNAUTHORIZED", "ERR_INVALID_PAYLOAD"
    pub message: String,    // Human-readable diagnostic description
    pub details: Option<serde_json::Value>,
}
```

---

### 2. Application State & Discovery Contracts

#### 2.1 `REQ_GET_APP_STATE` / `RESP_APP_STATE`
- **Contract ID**: `CTR-IPC-001`
- **Sender**: `kinetiwall-ui.exe` | **Receiver**: `kinetiwall-core.exe`
- **Purpose**: Retrieve global engine health, running player processes, and monitor assignments.
- **Payload (`REQ`)**: `{}` (empty object).
- **Payload (`RESP`)**:
  ```json
  {
    "engine_version": "1.0.0",
    "uptime_seconds": 3420,
    "active_monitors": [
      {
        "monitor_id": "\\\\.\\DISPLAY1",
        "edid_hash": "a8f3...12",
        "assigned_wallpaper_id": "550e8400-e29b-41d4-a716-446655440000",
        "player_pid": 14208,
        "playback_state": "playing"
      }
    ],
    "policy_state": { "fullscreen_active": false, "on_battery": false }
  }
  ```

#### 2.2 `REQ_SET_CONFIG` / `RESP_SET_CONFIG`
- **Contract ID**: `CTR-IPC-002`
- **Sender**: `kinetiwall-ui.exe` | **Receiver**: `kinetiwall-core.exe`
- **Purpose**: Persist user updates to application settings.
- **Authorization**: Validated against `KinetiWallConfig` schema. Reject unknown keys if validation fails.
- **Payload (`REQ`)**: Full or partial configuration patch conforming to `KW-DAT-004`.
- **Validation**: Strict boundary checks (framerate $15 \le fps \le 240$, volume $0.0 \le v \le 1.0$).
- **Success Response**: `{"status": "applied", "schema_version": 1}`.
- **Error Codes**: `ERR_SCHEMA_VIOLATION`, `ERR_OUT_OF_BOUNDS`, `ERR_PERSISTENCE_FAILED`.

---

### 3. Wallpaper Assignment Contracts

#### 3.1 `REQ_ASSIGN_WALLPAPER` / `RESP_ASSIGN_WALLPAPER`
- **Contract ID**: `CTR-IPC-003`
- **Sender**: `kinetiwall-ui.exe` | **Receiver**: `kinetiwall-core.exe`
- **Purpose**: Bind an installed wallpaper to a specific physical monitor.
- **Payload (`REQ`)**:
  ```json
  {
    "monitor_id": "\\\\.\\DISPLAY1",
    "wallpaper_id": "550e8400-e29b-41d4-a716-446655440000",
    "stretch_mode": "fill"
  }
  ```
- **Validation**:
  1. `monitor_id` must match an active display in `MonitorManager`.
  2. `wallpaper_id` must resolve to an installed package in `LibraryRepository`.
- **Side Effects**: Core spawns or reconfigures the player subprocess on that display, updates `config.json` atomically, and broadcasts `EVT_WALLPAPER_ASSIGNED`.

---

### 4. Playback Control Contracts

#### 4.1 `CMD_PLAYBACK_CONTROL` / `RESP_PLAYBACK_ACK`
- **Contract ID**: `CTR-IPC-004`
- **Sender**: `kinetiwall-core.exe` | **Receiver**: Player Processes (`kinetiwall-player-*`)
- **Purpose**: Direct instruction to change playback state.
- **Payload (`CMD`)**:
  ```json
  {
    "action": "pause",
    "reason": "policy_fullscreen_app",
    "fade_duration_ms": 150
  }
  ```
- **Allowed Actions**: `"play"`, `"pause"`, `"stop"`, `"mute"`, `"unmute"`.
- **Response**: `{"status": "acknowledged", "current_state": "paused", "frame_timestamp_ms": 14205}`.

---

### 5. Player Lifecycle Contracts

#### 5.1 `CMD_INIT_PLAYER` / `RESP_INIT_PLAYER`
- **Contract ID**: `CTR-IPC-005`
- **Sender**: `kinetiwall-core.exe` | **Receiver**: Newly spawned Player Process
- **Purpose**: Handshake initialization containing target monitor geometry and media entrypoint.
- **Payload (`CMD`)**:
  ```json
  {
    "player_id": "550e8400-e29b-41d4-a716-446655440000",
    "monitor_id": "\\\\.\\DISPLAY1",
    "window_bounds": { "x": 0, "y": 0, "width": 2560, "height": 1440 },
    "workerw_hwnd": 196854,
    "entrypoint_path": "C:\\Users\\...\\wallpapers\\{id}\\assets\\loop.mp4",
    "initial_volume": 0.8,
    "is_muted": false
  }
  ```
- **Response**: `{"status": "ready", "player_version": "1.0.0"}`.
- **Timeout**: 5000ms. If player does not respond with `status: ready`, Core marks it unresponsive, kills the PID, and attempts recovery.

#### 5.2 `EVT_PLAYER_CRASHED`
- **Contract ID**: `CTR-IPC-006`
- **Sender**: `kinetiwall-core.exe` (LifecycleSupervisor) | **Receiver**: `kinetiwall-ui.exe`
- **Purpose**: Notify UI that a player crashed and indicate recovery action (restart backoff count).

---

### 6. Customization & Audio Visualization Contracts

#### 6.1 `CMD_SYNC_PROPERTIES` / `RESP_SYNC_PROPERTIES`
- **Contract ID**: `CTR-IPC-007`
- **Sender**: `kinetiwall-core.exe` | **Receiver**: `kinetiwall-player-web.exe`
- **Purpose**: Deliver resolved dynamic properties to the active web wallpaper.
- **Payload**:
  ```json
  {
    "properties": {
      "clock_color": "#FF0055",
      "speed_multiplier": 1.2,
      "show_weather": true
    }
  }
  ```
- **Validation**: Values must conform to `PropertyDefinition` types in package manifest.

#### 6.2 `EVT_AUDIO_SPECTRUM` (Audio-Reactive Stream)
- **Contract ID**: `CTR-IPC-008`
- **Sender**: `kinetiwall-core.exe` (AudioVisualizationManager) | **Receiver**: `kinetiwall-player-web.exe`
- **Purpose**: Stream real-time frequency spectrum buckets for visualizers.
- **Frequency**: Configurable, capped at 30 Hz or 60 Hz.
- **Privacy & Security Constraint**: **Zero Raw Audio**. The payload contains strictly normalized float frequency buckets.
- **Payload**:
  ```json
  {
    "timestamp_ms": 1731234567890,
    "bucket_count": 16,
    "frequencies": [0.05, 0.12, 0.45, 0.82, 0.95, 0.70, 0.40, 0.22, 0.15, 0.10, 0.08, 0.05, 0.02, 0.01, 0.0, 0.0]
  }
  ```
- **Delivery Policy**: Dropped if the player pipe buffer is saturated; never queued indefinitely (lossy delivery for real-time telemetry).

---

### 7. Protocol Versioning and Deprecation Strategy

1. **Protocol Identifier**: `protocol_version: 1` in envelope header.
2. **Version Compatibility Rule**:
   - Clients and Servers check `protocol_version` on every received envelope.
   - If `protocol_version > 1`: Core rejects with `ERR_UNSUPPORTED_PROTOCOL_VERSION` and terminates the pipe session.
   - If `protocol_version < 1`: Legacy message is rejected.
3. **Additive Changes (Minor Evolution)**:
   - Optional fields added to existing message payloads do NOT bump `protocol_version`.
   - Consumers MUST safely ignore unknown fields (`#[serde(default)]`, `deny_unknown_fields` is omitted).

