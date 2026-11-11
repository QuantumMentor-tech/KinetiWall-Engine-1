# Windows API Compatibility Decisions & Risk Register
**Document ID:** `KW-FFI-006`  
**Deliverable:** F (Native API Compatibility and Open Decisions)  
**Parent Document:** [README.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/native_api_abstraction/README.md)  
**Classification:** Core System Architecture / Operational Reliability Standard  

---

## 1. Native API & FFI Risk Register

Interfacing safe Rust with legacy C-style Windows APIs introduces inherent technical risks:

| Risk ID | Risk Summary & Failure Mode | Prob. | Impact | Severity | Architecture Mitigation & Prevention Strategy |
| :--- | :--- | :---: | :---: | :---: | :--- |
| **`RSK-FFI-01`** | **Panic Across Foreign ABI Boundary**<br>A Rust panic in a native callback (e.g. `MonitorEnumProc`) unwinds across `extern "system"`, aborting the process. | Med | High | **CRITICAL** | **Mandatory Panic Barriers (`DEC-FFI-01`)**: All native callbacks wrap their inner logic in `std::panic::catch_unwind`, logging errors and returning safe sentinel values. |
| **`RSK-FFI-02`** | **Lost LastError via Intermediate Calls**<br>Calling a logging or memory allocation function before `GetLastError()` overwrites the thread's error state with `0`. | High | Med | **HIGH** | **Immediate Error Capture Pattern (`DEC-FFI-02`)**: `capture_last_win32_error()` must be invoked immediately after a failed API call; no intermediate calls permitted. |
| **`RSK-FFI-03`** | **Thread Affinity Violation on HWND**<br>Invoking window management functions from a background async worker thread induces deadlocks or invalid handle errors. | Med | High | **HIGH** | **Type-System Thread Affinity (`DEC-FFI-03`)**: `SafeHwnd` does *not* implement `Send` or `Sync`. Cross-thread window operations must dispatch through message loops. |
| **`RSK-FFI-04`** | **Handle Leak via Early Return**<br>A raw `HANDLE` or `HHOOK` is not released when an early return or `?` error operator triggers in a function. | High | Med | **HIGH** | **Universal RAII Drop Wrappers (`DEC-FFI-04`)**: All owned handles must be wrapped in `SafeHandle`, `SafeJobObject`, or `SafeHook` with deterministic `Drop` implementations. |
| **`RSK-FFI-05`** | **Null Byte Injection in UTF-16 Strings**<br>A string containing an interior null byte causes Win32 APIs to truncate file paths or pipe names, risking security bypass. | Low | High | **HIGH** | **Interior Null Rejection (`DEC-FFI-05`)**: `to_wide_null_terminated()` checks for `\0` and rejects strings containing embedded nulls with `PlatformError::InvalidParameter`. |
| **`RSK-FFI-06`** | **Accidental Destruction of Shell HWND**<br>Developer mistakenly calls `CloseHandle()` or `DestroyWindow()` on an externally discovered `WorkerW` or `Progman` HWND. | Med | High | **CRITICAL** | **Strict Handle Typology (`DEC-FFI-06`)**: Borrowed shell windows are wrapped in `ValidatedHostWindow` which exposes *no* drop or destruction methods. |
| **`RSK-FFI-07`** | **Unbounded Output Buffer Allocation**<br>A native query API returning required buffer length is exploited to trigger massive memory allocations. | Low | Med | **MEDIUM** | **Hard Buffer Allocation Ceiling (`DEC-FFI-07`)**: Dynamic buffer resizing enforces an absolute ceiling of $32\text{ KB}$. Beyond that, the call fails with `ResourceExhausted`. |
| **`RSK-FFI-08`** | **Mismatched Struct Padding / Alignment**<br>Manually declared Win32 structures have incorrect field offsets, leading to memory corruption during OS calls. | Low | High | **CRITICAL** | **Official `windows` Crate Bindings (`DEC-FFI-08`)**: All structure layouts must originate from Microsoft's authoritative `windows` crate (v0.58). Manual struct layouts are prohibited. |

---

## 2. Architecture Decision Records (Native API & FFI: `DEC-FFI-01` to `08`)

The following formal decisions govern all Win32 FFI and platform abstraction code across KinetiWall:

### `DEC-FFI-01`: Mandatory `catch_unwind` on All Foreign ABI Callbacks
* **Context:** Rust panics unwinding across foreign C-ABIs trigger an unrecoverable process abort (`std::process::abort`), preventing cleanup handlers from running.
* **Decision:** Every `extern "system"` callback implemented by KinetiWall must be wrapped in `std::panic::catch_unwind`.
* **Consequences:** Eliminates abort hazards. Caught panics log a critical diagnostic event and return a documented failure sentinel (`FALSE` or `0`).

### `DEC-FFI-02`: Immediate Win32 Error Capture via Dedicated Helper
* **Context:** `GetLastError()` context is fragile and easily overwritten by subsequent memory allocations, logging calls, or standard library helpers.
* **Decision:** Implement `capture_last_win32_error(operation)` which immediately queries `GetLastError()` and converts it to a `PlatformError`.
* **Consequences:** Preserves 100% of native error forensics for debugging and automated recovery.

### `DEC-FFI-03`: Enforce `!Send` and `!Sync` on Window Handles (`SafeHwnd`)
* **Context:** Win32 window handles (`HWND`) are inherently thread-affine. Sharing them arbitrarily across async threads causes message queue desynchronization.
* **Decision:** `SafeHwnd` uses `PhantomData<*mut ()>` to prevent `Send` and `Sync` implementations. Window operations are strictly bound to the thread owning the message loop.
* **Consequences:** Rust compiler statically prevents cross-thread window manipulation bugs at compile time.

### `DEC-FFI-04`: Universal RAII Drop Semantics for Owned Kernel Resources
* **Context:** Manual handle closing via `CloseHandle()` is error-prone, leading to handle table exhaustion over prolonged runtime.
* **Decision:** All owned kernel objects are encapsulated in RAII wrappers (`SafeHandle`, `SafeJobObject`, `SafePowerSubscription`).
* **Consequences:** Guarantees zero handle leaks during normal execution, early error returns (`?`), or panics.

### `DEC-FFI-05`: Strict Interior Null Checking on All UTF-16 Conversions
* **Context:** Win32 UTF-16 APIs interpret the first `0u16` as the string terminator. Passing strings with interior nulls can truncate security tokens, pipe names, or paths.
* **Decision:** Reject any Rust string containing an interior `\0` byte with `PlatformError::InvalidParameter`.
* **Consequences:** Prevents path-truncation security vulnerabilities and null-byte injection attacks.

### `DEC-FFI-06`: Explicit Typology Demarcation (Owned Handles vs Borrowed HWNDs)
* **Context:** Applying `CloseHandle` to an `HWND` is invalid, while destroying external shell windows crashes Windows Explorer.
* **Decision:** Establish distinct types: `SafeHandle` (owned, drops with `CloseHandle`), `SafeHwnd` (owned, drops with `DestroyWindow`), and `ValidatedHostWindow` (borrowed, no drop destructor).
* **Consequences:** Enforces correct release semantics via the type system; impossible to accidentally close a borrowed shell window.

### `DEC-FFI-07`: Bounded Buffer Resizing for Native String Queries
* **Context:** Querying variable-length strings from Windows could theoretically be manipulated to trigger out-of-memory panics.
* **Decision:** Enforce a hard cap of $32\text{ KB}$ (16,384 `u16` characters) on any dynamically resized output buffer.
* **Consequences:** Completely prevents integer overflow and memory exhaustion during string queries.

### `DEC-FFI-08`: Authoritative Adoption of `windows` Crate (v0.58)
* **Context:** Deprecated crates like `winapi` suffer from stale struct definitions, lack of ARM64 support, and missing modern Windows 11 APIs.
* **Decision:** Adopt Microsoft's official `windows` crate (v0.58) as the sole authoritative Win32 binding dependency (`DEP-01`), supplemented by `windows-sys` where minimal FFI overhead is required.
* **Consequences:** Guarantees 100% accurate struct layouts, full Windows 10/11 API coverage, active Microsoft maintenance, and seamless ARM64 support.

---

## 3. Open Engineering Review Items

| Item ID | Description | Current Architectural Status | Target Phase |
| :---: | :--- | :--- | :---: |
| **`REV-FFI-01`** | **High-Precision QueryPerformanceCounter Wrapper:** Determine whether a dedicated monotonic timer wrapper is needed for sub-millisecond frame pacing in mpv D3D11 rendering. | Rust's `std::time::Instant` uses `QPC` internally on Windows; evaluating whether raw `QPC` offers measurable benefits. | Phase 4 (Runtime Verification) |
| **`REV-FFI-02`** | **DWM Composition Timing Hooks:** Investigate whether `DwmGetCompositionTimingInfo` should be abstracted to synchronize wallpaper frame presentation with Windows DWM VSync. | Trait interface specified; concrete DWM timing wrapper to be evaluated during player rendering implementation. | Phase 4 (Runtime Verification) |
| **`REV-FFI-03`** | **Miri FFI Limitations on Windows:** Evaluate which Win32 mock wrappers can be fully validated under Miri vs requiring live Windows test VMs. | Pure logic (string conversion, error mapping, coordinate math) runs under Miri; Win32 APIs tested in VM. | Phase 4 (Runtime Verification) |

