# Native Windows API Abstraction Test Plan
**Document ID:** `KW-FFI-005`  
**Deliverable:** E (Native API Verification and Test Plan)  
**Parent Document:** [README.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/native_api_abstraction/README.md)  
**Classification:** Quality Assurance & System Verification Specification  

---

## 1. Test Strategy & Acceptance Framework

Testing the native Win32 API abstraction layer presents unique architectural challenges: native Win32 calls interact directly with the operating system kernel and desktop shell. Testing must guarantee safety without corrupting developer workstations or requiring physical multi-monitor setups for continuous integration (CI).

Testing is structured into five distinct levels:
1. **Tier 1: Pure Rust Unit Tests** (No Win32 API calls; tests error mapping, string conversion, buffer sizing, and coordinate math).
2. **Tier 2: Mock Trait Integration Tests** (Uses `MockDisplayProvider`, `MockShellIntegrator` to test application logic under fault injection).
3. **Tier 3: Windows Integration Harness** (Runs in an isolated Windows environment; creates dummy test windows and validates `SetParent`, DPI, and handle lifecycles).
4. **Tier 4: Unsafe Code Audit Review** (Automated static analysis with `cargo-geiger`, Miri, and manual safety-comment verification).
5. **Tier 5: Performance & Overhead Benchmarks** (Validates latency and memory retention of API wrappers).

> [!IMPORTANT]
> All test cases below represent approved verification specifications. Their status is marked as `NOT RUN (DESIGN PHASE)` pending runtime implementation in Phase 4.

---

## 2. Acceptance Criteria Verification Matrix (`FFI-001` through `FFI-012`)

| Test ID | Test Title | Primary Target & Scope | Expected Acceptance Criteria | Status |
| :--- | :--- | :--- | :--- | :--- |
| **`FFI-001`** | **SafeJobObject Teardown** | `SafeJobObject` / `Drop` | Dropping a `SafeJobObject` closes the underlying kernel handle; kernel terminates all assigned child processes within $\le 50\text{ ms}$. | `NOT RUN (DESIGN PHASE)` |
| **`FFI-002`** | **SafeHandle RAII Leak Prevention** | `SafeHandle` / `Drop` | Creating and dropping 1,000 process handles in a loop results in zero handle leaks (`GetProcessHandleCount` $\Delta = 0$). | `NOT RUN (DESIGN PHASE)` |
| **`FFI-003`** | **SingleInstanceGuard Release** | Win32 Named Mutex | Dropping `SingleInstanceGuard` releases the mutex; subsequent `CreateMutexW` call succeeds immediately with `ERROR_SUCCESS`. | `NOT RUN (DESIGN PHASE)` |
| **`FFI-004`** | **Named Pipe UTF-16 Safety** | Pipe Name Conversion | String with embedded null byte (`\\.\pipe\test\0name`) is rejected with `InvalidParameter`; valid names convert to null-terminated UTF-16. | `NOT RUN (DESIGN PHASE)` |
| **`FFI-005`** | **SafeHwnd Thread Affinity** | `SafeHwnd` (`!Send`, `!Sync`)| Attempting to transfer `SafeHwnd` across thread boundaries fails at compile time (`Send` trait not implemented). | `NOT RUN (DESIGN PHASE)` |
| **`FFI-006`** | **GetLastError Immediate Capture** | `capture_last_win32_error` | Simulating an API failure captures the exact OS error code (`ERROR_FILE_NOT_FOUND`) even if memory allocations occur during error wrapping. | `NOT RUN (DESIGN PHASE)` |
| **`FFI-007`** | **Foreign Callback Panic Barrier** | `catch_unwind` in Callbacks | Panicking inside a monitor enumeration callback is caught cleanly; returns `FALSE` without crashing or aborting the host process. | `NOT RUN (DESIGN PHASE)` |
| **`FFI-008`** | **Negative Coordinate Normalization**| Display Math Engine | Secondary monitor at $[-1920, 0]$ with virtual screen at $[-1920, 0]$ normalizes child window coordinates to $[0, 0]$ relative to `WorkerW`. | `NOT RUN (DESIGN PHASE)` |
| **`FFI-009`** | **WorkerW Dual-Key Validation** | `revalidate_host_handle` | Window with class other than `"WorkerW"` or owned by a non-explorer PID is rejected with `PlatformError::UnexpectedState`. | `NOT RUN (DESIGN PHASE)` |
| **`FFI-010`** | **DPI Scaling Calculations** | Per-Monitor V2 DPI API | Querying 150% scaling display returns scale factor $1.5$; window sizing multiplies base bounds accurately without integer truncation. | `NOT RUN (DESIGN PHASE)` |
| **`FFI-011`** | **Explorer Crash Re-anchoring** | Mock Shell Integrator | Injected host invalidation triggers `RECONCILIATION_PENDING`; circuit breaker limits re-anchoring to 3 retries in 60 seconds. | `NOT RUN (DESIGN PHASE)` |
| **`FFI-012`** | **Zero-Cost Abstraction Latency** | Safe Wrapper Benchmarks | Executing `SafeHandle::new`, `SafeHwnd::new_owned`, and error captures incurs $< 15\text{ ns}$ overhead compared to raw Win32 calls. | `NOT RUN (DESIGN PHASE)` |

---

## 3. Mockable Test Double Architecture (Tier 2)

To enable comprehensive unit testing of higher-level engine services without depending on live Windows User32/GDI subsystems, KinetiWall provides full-featured mock implementations:

```rust
#[cfg(test)]
pub struct MockDisplayProvider {
    pub displays: std::sync::Mutex<Vec<DisplayDescriptor>>,
}

#[cfg(test)]
impl DisplayProvider for MockDisplayProvider {
    fn query_displays(&self) -> Result<Vec<DisplayDescriptor>, PlatformError> {
        Ok(self.displays.lock().unwrap().clone())
    }

    fn get_display_dpi(&self, _display_id: &str) -> Result<u32, PlatformError> {
        Ok(96) // Standard 100% DPI
    }
}
```

Higher-level unit tests inject `MockDisplayProvider` and `MockShellIntegrator` into the engine controller, allowing simulated testing of:
* Monitor disconnection and dynamic resolution resizing.
* Sudden Explorer restarts and host invalidations.
* Win32 API timeouts and permission-denied errors.

---

## 4. Automated Unsafe Code Auditing Checklist & Tooling

To guarantee ongoing compliance with the Rust FFI Safety Policy, the continuous integration (CI) pipeline executes automated checks:

1. **Static Unsafe Audit (`cargo geiger`):** Scans the codebase to ensure `unsafe` blocks are strictly contained within `kinetiwall_platform::win32::raw`. Any `unsafe` block in core or player modules fails the build.
2. **Undefined Behavior Detection (Miri):** Runs unit tests under Miri (`cargo miri test`) to detect unaligned pointer access, memory leaks, invalid UTF-16 conversions, or stacked borrows violations.
3. **Safety Comment Enforcement Script:** A custom pre-commit script verifies that every `unsafe` keyword in the source tree is immediately preceded by an auditable `// SAFETY:` comment block.

