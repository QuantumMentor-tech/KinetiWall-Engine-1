# Rust FFI & Unsafe Code Safety Policy
**Document ID:** `KW-FFI-002`  
**Deliverable:** B (Rust FFI and Unsafe-Code Policy)  
**Parent Document:** [README.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/native_api_abstraction/README.md)  
**Classification:** Unsafe Rust Safety Standard / Secure Coding Guidelines  

---

## 1. Executive Summary & Policy Scope

Rust guarantees memory safety, data-race freedom, and thread safety in safe code. However, interacting with native Windows APIs via FFI requires crossing into `unsafe` Rust.

This policy establishes strict, non-negotiable rules for all unsafe code within KinetiWall:
1. Minimizing the unsafe surface area to the lowest practical boundary.
2. Requiring structured, auditable `// SAFETY:` rationale blocks for every unsafe operation.
3. Guaranteeing pointer, buffer, and struct alignment invariants.
4. Enforcing thread-affinity constraints and prohibiting illegal `Send` / `Sync` implementations.
5. Preventing Rust panics from crossing foreign C-ABI boundaries (`extern "system"`).

---

## 2. Mandatory Unsafe Code Invariant Comments (`// SAFETY:`)

In accordance with Rust best practices, every `unsafe` block, function, or trait implementation in KinetiWall **must** be immediately preceded by a structured `// SAFETY:` comment block answering five explicit questions:

```rust
// SAFETY:
// 1. Preconditions: `h_wnd` has been validated with `IsWindow` and belongs to current session.
// 2. Pointer Validity: `rect_out` points to an aligned, initialized `RECT` on the local stack.
// 3. Ownership / Lifetime: Windows does not retain the pointer after `GetWindowRect` returns.
// 4. Thread Affinity: Function is called from the thread that created the window or owns the message queue.
// 5. Failure Capture: Return value is checked; if 0, `GetLastError()` is captured immediately.
unsafe {
    let res = GetWindowRect(h_wnd, rect_out);
    if res == 0 {
        return Err(capture_last_win32_error("GetWindowRect"));
    }
}
```

Code reviews and automated CI linters will reject any pull request containing an `unsafe` block without this complete 5-point verification comment.

---

## 3. String & Buffer Safety Rules

Win32 APIs commonly use null-terminated UTF-16 wide strings (`PCWSTR`, `PWSTR`) and caller-allocated output buffers. Incorrect handling leads to buffer overflows, truncated paths, and undefined behavior.

### 3.1 Rust UTF-8 to Windows UTF-16 (`PCWSTR`)

1. **Embedded Null Detection:** Any input string containing an interior null byte (`\0`) must be rejected immediately with `PlatformError::InvalidParameter("String contains interior null byte")`.
2. **Deterministic Null-Termination:** UTF-16 encoding must explicitly append a terminating `0u16`:
   ```rust
   pub fn to_wide_null_terminated(s: &str) -> Result<Vec<u16>, PlatformError> {
       if s.contains('\0') {
           return Err(PlatformError::InvalidParameter("Interior null byte"));
       }
       Ok(s.encode_utf16().chain(std::iter::once(0)).collect())
   }
   ```
3. **Lifetime Guarantee:** The backing `Vec<u16>` must outlive the `PCWSTR` pointer passed to the Win32 API. Never pass temporary expressions like `to_wide(s).as_ptr()` to an API that stores or uses the pointer asynchronously.

### 3.2 Output Buffers & Variable-Length Win32 Queries

When querying strings from Windows (e.g. `RealGetWindowClassW` or `QueryFullProcessImageNameW`):
1. **Fixed Buffers:** Use pre-allocated, sized stack arrays with explicit bounds:
   ```rust
   let mut buffer = [0u16; 256];
   // SAFETY: Buffer is stack-allocated, aligned, and length is passed accurately.
   let len = unsafe { RealGetWindowClassW(hwnd, &mut buffer) };
   if len == 0 {
       return Err(capture_last_win32_error("RealGetWindowClassW"));
   }
   let class_name = String::from_utf16_lossy(&buffer[..len as usize]);
   ```
2. **Dynamic Buffers:** When an API indicates a buffer is too small (e.g. `ERROR_INSUFFICIENT_BUFFER` or returning required size), double the buffer size up to an enforced maximum ceiling ($32\text{ KB}$). Never allocate unbounded memory based on raw integer values returned from the OS.

---

## 4. Windows Structure Initialization & Alignment

Win32 structures (such as `JOBOBJECT_EXTENDED_LIMIT_INFORMATION`, `MONITORINFOEXW`, `STARTUPINFOW`) frequently require explicit size initialization and specific memory alignment:

1. **Explicit Zero Initialization:** All structures must be initialized using `std::mem::zeroed()` or `Default::default()`. Never leave padding bytes uninitialized.
2. **Mandatory Size Fields (`cb` / `cbSize`):** Many Windows structures include a self-sizing header field that must match the exact compiled struct size:
   ```rust
   let mut info: MONITORINFOEXW = unsafe { std::mem::zeroed() };
   info.monitorInfo.cbSize = std::mem::size_of::<MONITORINFOEXW>() as u32;
   ```
3. **Structure Packing:** Always use struct definitions provided by the official `windows` crate. Never redefine native structures manually with arbitrary packing attributes unless validating against Microsoft SDK headers.

---

## 5. Thread Affinity & `Send` / `Sync` Rules

Windows UI and windowing APIs are strictly **thread-affine**:
* A window's message queue is tied to the thread that invoked `CreateWindowExW`.
* Calling window-management functions from unrelated threads without synchronization causes deadlocks, missed messages, or `ERROR_INVALID_WINDOW_HANDLE`.

### 5.1 Handle Concurrency Rules

| Type | Native Type | `Send`? | `Sync`? | Justification & Architecture Constraint |
| :--- | :--- | :---: | :---: | :--- |
| **`SafeHwnd`** | `HWND` | **No** | **No** | **Thread-Affine**: Must remain bound to the thread that created the window or manages its message loop (`!Send`, `!Sync`). |
| **`SafeHandle`** | `HANDLE` (Kernel) | **Yes** | **No** | Kernel handles (Process, Mutex, Job Object) can be passed between threads, but concurrent multi-thread mutation requires internal locking. |
| **`SafeMonitor`**| `HMONITOR` | **Yes** | **Yes** | Borrowed display handle; immutable metadata query target; thread-safe across Windows threads. |
| **`SafeHook`** | `HWINEVENTHOOK` | **No** | **No** | Tied to the thread's Win32 message pump that registered the hook. |

> [!CAUTION]
> Never implement `unsafe impl Send` or `unsafe impl Sync` on `SafeHwnd`. If another thread needs to interact with a window, it must dispatch an asynchronous message via `PostMessageW` or a channel to the owning thread's message loop.

---

## 6. Panic Barriers at Foreign FFI Boundaries

In Rust, unwinding a panic across a foreign C ABI boundary (`extern "system"`) results in **immediate process abort** (`std::process::abort`). This is catastrophic because it prevents RAII drop handlers, database locks, and Job Object cleanup from executing.

### 6.1 Native Callback Safety Wrapper

All native Windows callbacks (e.g. `EnumWindowsProc`, `MonitorEnumProc`, `WinEventProc`) must enforce a complete panic barrier using `std::panic::catch_unwind`:

```rust
pub unsafe extern "system" fn monitor_enum_proc(
    h_monitor: HMONITOR,
    h_dc: HDC,
    rect: *mut RECT,
    l_param: LPARAM,
) -> BOOL {
    // Enforce strict panic barrier: catch unwind and prevent C-ABI abort
    let result = std::panic::catch_unwind(std::panic::AssertUnwindSafe(|| {
        // Safe callback logic here...
        BOOL(1)
    }));

    match result {
        Ok(bool_val) => bool_val,
        Err(_) => {
            // Log critical panic and return FALSE to abort enumeration safely
            eprintln!("[CRITICAL] Panic caught in monitor_enum_proc; aborting enumeration");
            BOOL(0)
        }
    }
}
```

---

## 7. Unsafe Code Review Checklist

Before any pull request introducing unsafe code can be merged into KinetiWall, it must be verified against this 8-point checklist:

- [ ] **Minimal Scope:** Is the unsafe block narrowed to only the exact Win32 call, with preprocessing and postprocessing in safe Rust?
- [ ] **Safety Comment:** Does every unsafe block contain an auditable 5-point `// SAFETY:` rationale comment?
- [ ] **Pointer Validity:** Are raw pointers checked for null, proper alignment, and lifetime validity?
- [ ] **Struct Sizing:** Are native struct size fields (`cbSize`) initialized correctly using `mem::size_of`?
- [ ] **Immediate Error Capture:** Is `GetLastError()` captured immediately if the API reports failure?
- [ ] **Thread Affinity:** Does the code respect the thread-affinity rules of the underlying Win32 API?
- [ ] **Zero Unwinding:** Are all foreign callbacks wrapped in `std::panic::catch_unwind`?
- [ ] **RAII Encapsulation:** Are all returned kernel handles wrapped in safe RAII types implementing `Drop`?

