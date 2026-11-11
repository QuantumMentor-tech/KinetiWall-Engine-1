# Windows API Error Mapping & Taxonomy
**Document ID:** `KW-FFI-004`  
**Deliverable:** D (Windows API Error Mapping Reference)  
**Parent Document:** [README.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/native_api_abstraction/README.md)  
**Classification:** System Diagnostics Reference / Error Handling Standard  

---

## 1. Native Win32 Error Reporting Complexity

Win32 APIs exhibit diverse and inconsistent error reporting mechanisms:
1. **Return Boolean + `GetLastError()`:** Returns `FALSE` (`0`) on failure; caller must inspect `GetLastError()`. (e.g. `SetWindowPos`, `CloseHandle`).
2. **Return Handle + Sentinel:** Returns `NULL` (`0`) or `INVALID_HANDLE_VALUE` (`-1`) on failure. (e.g. `CreateFileW`, `CreateJobObjectW`).
3. **Return Integer / Count:** Returns `0` on error, but `0` can also represent a valid empty result. (e.g. `RealGetWindowClassW`).
4. **Return HRESULT:** COM-style signed 32-bit integer where negative values denote failure. (e.g. D3D11, DXGI, WASAPI).
5. **No Error Reporting:** Legacy APIs that provide no error codes on failure.

Converting all these raw values into a single generic `Err("Failed")` string discards critical forensic information. KinetiWall implements a structured, typed mapping from native Windows error codes to domain-specific `PlatformError` types.

---

## 2. Immediate Error Capture Rule

`GetLastError()` inspects the calling thread's Thread Information Block (TIB). Many innocent operations (such as memory allocations, string formatting, or tracing calls) invoke internal Win32 APIs that overwrite this thread-local state.

```rust
// Architectural rule for immediate error capture
#[inline(always)]
pub fn capture_last_win32_error(operation: &'static str) -> PlatformError {
    // SAFETY: GetLastError is thread-safe and has no preconditions.
    let code = unsafe { GetLastError() };
    PlatformError::from_win32(operation, code.0)
}
```

**Rule:** `GetLastError()` must be called immediately on the line following the failed API call.

---

## 3. Native Win32 to Application Error Mapping Matrix

The following table defines the formal taxonomy and mapping between Windows system error codes and KinetiWall application error categories:

| Win32 Error Code | Constant Name | KinetiWall Domain Error Variant | Error Category | Retry Eligible? | Recovery Action | Public UI Message |
| :---: | :--- | :--- | :--- | :---: | :--- | :--- |
| `0x00000002` | `ERROR_FILE_NOT_FOUND` | `PlatformError::ResourceNotFound` | Permanent | **No** | Verify installation paths; check binary integrity. | "Requested system file or wallpaper binary is missing." |
| `0x00000005` | `ERROR_ACCESS_DENIED` | `PlatformError::AccessDenied` | Security / Perms | **No** | Log DACL violation; verify user token; do not elevate. | "Permission denied. Windows blocked access to the desktop surface." |
| `0x00000006` | `ERROR_INVALID_HANDLE` | `PlatformError::InvalidHandle` | Lifecycle Bug | **No** | Invalidate cached handle; trigger re-discovery. | "Internal handle became invalid." |
| `0x0000000E` | `ERROR_OUTOFMEMORY` | `PlatformError::ResourceExhausted` | Transient / Crit | **Yes** (Backoff) | Drop wallpaper caches; restart player with reduced texture limits. | "System memory low. KinetiWall reduced resource usage." |
| `0x00000057` | `ERROR_INVALID_PARAMETER` | `PlatformError::InvalidParameter` | Programmer Error | **No** | Fatal invariant violation; report bug in debug build. | "An invalid parameter was passed to the Windows subsystem." |
| `0x00000578` | `ERROR_INVALID_WINDOW_HANDLE`| `PlatformError::WindowDestroyed` | External Shell | **Yes** (Re-anchor)| Invalidate host reference; trigger `RECONCILIATION_PENDING`. | "Desktop window closed by Explorer. Re-anchoring..." |
| `0x000005B4` | `ERROR_TIMEOUT` | `PlatformError::Timeout` | Operational | **Yes** (Circuit) | If shutdown: escalate to `TerminateProcess`. If discovery: retry. | "Windows desktop shell did not respond in time." |
| `0x000000B7` | `ERROR_ALREADY_EXISTS` | `PlatformError::AlreadyExists` | Concurrency | **No** | For Mutex: signal running instance and exit. | "Another instance of KinetiWall is already running." |
| `0x000000E8` | `ERROR_NO_DATA` | `PlatformError::PipeBroken` | IPC Lifecycle | **No** | Mark child process disconnected; poll exit code. | "Communication pipe closed by subprocess." |

---

## 4. Structured Error Representation (`PlatformError`)

The `PlatformError` enum encapsulates complete diagnostic context while remaining friendly to safe Rust pattern matching:

```rust
#[derive(Debug, thiserror::Error)]
pub enum PlatformError {
    #[error("Win32 API '{operation}' failed with OS error {win32_code}: {description}")]
    Win32Failure {
        operation: &'static str,
        win32_code: u32,
        description: String,
    },

    #[error("Desktop host validation failed: {0}")]
    HostValidationFailed(String),

    #[error("Window handle {0:?} was invalidated externally")]
    HandleInvalidated(&'static str),

    #[error("Resource handle recycled by unrelated process")]
    HandleRecycled(&'static str),

    #[error("Native operation '{0}' timed out")]
    Timeout(&'static str),

    #[error("Permission denied during '{0}'")]
    AccessDenied(&'static str),

    #[error("Invalid parameter: {0}")]
    InvalidParameter(&'static str),

    #[error("Required system resource exhausted: {0}")]
    ResourceExhausted(&'static str),

    #[error("Unexpected platform state: {0}")]
    UnexpectedState(String),
}
```

### 4.1 Telemetry Serialization

When a `PlatformError` occurs, it automatically serializes into an auditable structured diagnostic event:

```json
{
  "timestamp": "2026-11-11T11:18:40.102Z",
  "level": "ERROR",
  "component": "kinetiwall-platform",
  "subsystem": "win32::shell",
  "event_type": "WIN32_API_FAILURE",
  "payload": {
    "operation": "SendMessageTimeoutW",
    "target_hwnd": "0x0001004A",
    "win32_code": 1460,
    "constant_name": "ERROR_TIMEOUT",
    "description": "This operation returned because the timeout period expired.",
    "retry_eligible": true,
    "recovery_action": "RETRY_WITH_BACKOFF"
  }
}
```

