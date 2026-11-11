# Native Resource Ownership & RAII Matrix
**Document ID:** `KW-FFI-003`  
**Deliverable:** C (Native Resource Ownership and RAII Matrix)  
**Parent Document:** [README.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/native_api_abstraction/README.md)  
**Classification:** System Architecture Reference / Resource Ownership Standard  

---

## 1. Native Resource Ownership Typology

Operating system resources on Windows fall into three distinct ownership categories:

1. **Owned Kernel Objects (`HANDLE`):** Created via kernel APIs (`CreateJobObjectW`, `CreateProcessW`, `CreateNamedPipeW`, `CreateMutexW`). These consume kernel memory and handle table slots. **Mandatory release rule:** Must be closed via `CloseHandle()`.
2. **Window Identifiers (`HWND`):** Identifiers representing desktop windows created via `CreateWindowExW` or discovered via `FindWindowW`. **Mandatory release rule:** Owned windows created by the process must be destroyed via `DestroyWindow()`. Externally discovered windows (like `WorkerW` or `Progman`) are **borrowed handles**; calling `DestroyWindow` or `CloseHandle` on them is forbidden.
3. **Borrowed Platform Handles (`HMONITOR`, `HDC`):** Ephemeral references passed by Windows during callbacks or queries. **Mandatory release rule:** Must never be closed or retained beyond the callback scope.

---

## 2. RAII Drop Wrappers for Owned Resources

To ensure that resources are reclaimed deterministically during normal exit, early returns, or unwinding, KinetiWall encapsulates every owned resource in a zero-cost RAII type implementing `Drop`:

### 2.1 The Universal `SafeHandle` (Kernel Objects)

```rust
pub struct SafeHandle {
    raw: HANDLE,
}

impl SafeHandle {
    pub fn new(raw: HANDLE) -> Result<Self, PlatformError> {
        if raw.is_invalid() || raw.0 == 0 {
            return Err(PlatformError::InvalidHandle);
        }
        Ok(Self { raw })
    }

    pub fn as_raw(&self) -> HANDLE {
        self.raw
    }

    /// Detaches handle ownership without closing it
    pub fn into_raw(mut self) -> HANDLE {
        let raw = self.raw;
        self.raw = HANDLE::default();
        std::mem::forget(self);
        raw
    }
}

impl Drop for SafeHandle {
    fn drop(&mut self) {
        if !self.raw.is_invalid() && self.raw.0 != 0 {
            // SAFETY: Valid, non-null kernel handle owned exclusively by this SafeHandle.
            unsafe {
                CloseHandle(self.raw);
            }
        }
    }
}
```

### 2.2 The `SafeHwnd` (Process-Owned Windows)

```rust
pub struct SafeHwnd {
    raw: HWND,
    owning_thread_id: u32,
    _marker: std::marker::PhantomData<*mut ()>, // Enforces !Send and !Sync
}

impl SafeHwnd {
    pub fn new_owned(raw: HWND) -> Result<Self, PlatformError> {
        if raw.0 == 0 || !unsafe { IsWindow(raw) }.as_bool() {
            return Err(PlatformError::InvalidHandle);
        }
        Ok(Self {
            raw,
            owning_thread_id: unsafe { GetCurrentThreadId() },
            _marker: std::marker::PhantomData,
        })
    }
}

impl Drop for SafeHwnd {
    fn drop(&mut self) {
        if self.raw.0 != 0 && unsafe { IsWindow(self.raw) }.as_bool() {
            // SAFETY: Window was created by this thread and is an owned window.
            unsafe {
                DestroyWindow(self.raw);
            }
        }
    }
}
```

---

## 3. Comprehensive Native Resource Ownership Matrix

The following matrix catalogs all native Windows resources utilized across the KinetiWall architecture:

| Resource Identifier | Native Type | Owning Abstraction | Creation / Acquisition API | Owned vs Borrowed | Destruction / Release API | Thread Affinity | Release Failure Behavior | Related Tests |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **`RES-KRN-01`** | `HANDLE` (Job) | `SafeJobObject` | `CreateJobObjectW` | **Owned** | `CloseHandle` | Any Thread | Kernel leaks job until process termination | `FFI-001`, `LIFE-003` |
| **`RES-KRN-02`** | `HANDLE` (Process) | `SafeHandle` | `CreateProcessW` | **Owned** | `CloseHandle` | Any Thread | OS handle table slot retained | `FFI-002`, `LIFE-005` |
| **`RES-KRN-03`** | `HANDLE` (Mutex) | `SingleInstanceGuard` | `CreateMutexW` | **Owned** | `ReleaseMutex` + `CloseHandle` | Any Thread | Mutex abandoned; next launch fails | `FFI-003`, `LIFE-008` |
| **`RES-KRN-04`** | `HANDLE` (Pipe) | `NamedPipeListener` | `CreateNamedPipeW` | **Owned** | `DisconnectNamedPipe` + `CloseHandle`| Any Thread | Pipe name locked; client connect fails | `FFI-004`, `LIFE-005` |
| **`RES-WIN-01`** | `HWND` (Player) | `SafeHwnd` | `CreateWindowExW` | **Owned** | `ShowWindow(SW_HIDE)` + `DestroyWindow`| **Owning Thread Only** | Ghost surface; DWM compositing overhead | `FFI-005`, `DSK-013` |
| **`RES-WIN-02`** | `HWND` (Core Host) | `SafeHwnd` | `CreateWindowExW` | **Owned** | `DestroyWindow` | **Owning Thread Only** | Message pump stops receiving OS events | `FFI-006`, `LIFE-005` |
| **`RES-WIN-03`** | `HWND` (WorkerW) | `ValidatedHostWindow` | `FindWindowW` / `0x052C` | **Borrowed** | **NONE** (Never destroy external shell HWND)| Any Thread | If closed accidentally: Explorer crashes | `FFI-007`, `DSK-008` |
| **`RES-WIN-04`** | `HWND` (Progman) | `RawHwnd` | `FindWindowW` | **Borrowed** | **NONE** (Never destroy external shell HWND)| Any Thread | If closed accidentally: Explorer crashes | `FFI-007`, `DSK-008` |
| **`RES-NOT-01`** | `HPOWERNOTIFY` | `SafePowerSubscription` | `RegisterPowerSettingNotification` | **Owned** | `UnregisterPowerSettingNotification` | Owning Thread | Kernel memory leak; dangling callbacks | `FFI-008`, `LIFE-009` |
| **`RES-NOT-02`** | `HWINEVENTHOOK` | `SafeWinEventHook` | `SetWinEventHook` | **Owned** | `UnhookWinEvent` | **Owning Thread Only** | Global event hook leak; performance hit | `FFI-009`, `LIFE-005` |
| **`RES-DSP-01`** | `HMONITOR` | `SafeMonitor` | `EnumDisplayMonitors` | **Borrowed** | **NONE** (Never close display handle) | Any Thread | Closing returns invalid parameter | `FFI-010`, `DSK-006` |

---

## 4. Handle Invalidation & Re-Validation Protocol

A critical vulnerability in desktop integration is operating against a native handle that has been destroyed externally (e.g. `explorer.exe` crash or display disconnect):

```rust
pub fn revalidate_host_handle(h_target: HWND, expected_pid: u32) -> Result<(), PlatformError> {
    // Check 1: Is handle still recognized as a live window by User32?
    if !unsafe { IsWindow(h_target) }.as_bool() {
        return Err(PlatformError::HandleInvalidated("Window handle no longer exists"));
    }

    // Check 2: Does the window still belong to explorer.exe?
    let mut current_pid = 0u32;
    unsafe { GetWindowThreadProcessId(h_target, Some(&mut current_pid)) };
    if current_pid != expected_pid {
        return Err(PlatformError::HandleRecycled("HWND recycled by unrelated process"));
    }

    // Check 3: Is window class still WorkerW?
    let mut buffer = [0u16; 32];
    let len = unsafe { RealGetWindowClassW(h_target, &mut buffer) };
    let class = String::from_utf16_lossy(&buffer[..len as usize]);
    if class != "WorkerW" {
        return Err(PlatformError::UnexpectedState("Window class mutated from WorkerW"));
    }

    Ok(())
}
```

Any operation attempting to position or reparent into a `WorkerW` host must execute this revalidation immediately prior to the Win32 call.

