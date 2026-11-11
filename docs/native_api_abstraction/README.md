# Native Win32 API Abstraction, Rust FFI Safety & Error Contracts
**Document ID:** `KW-FFI-000`  
**Phase:** 4 (Windows Platform Integration & Desktop Hosting)  
**Step:** 17 of 90  
**Status:** Approved Architectural Specification  
**Classification:** Windows Platform Integration / Unsafe Rust Safety Standard  

---

## 1. Document Overview and Scope

This document suite establishes the authoritative engineering specification for KinetiWall's **Native Win32 API Abstraction Layer, Rust Foreign Function Interface (FFI) Safety Architecture, and Operating System Error Contracts**.

KinetiWall interacts deeply with the Windows NT kernel and desktop shell:
1. Enumerating displays, virtual-desktop geometry, and per-monitor DPI scaling.
2. Manipulating desktop shell hierarchies (`Progman`, `WorkerW`, `SHELLDLL_DefView`) and window messages (`0x052C`).
3. Managing native process lifecycles via Windows Job Objects (`JOB_OBJECT_LIMIT_KILL_ON_JOB_CLOSE`).
4. Reparenting child DirectX 11 / mpv / WebView2 render surfaces (`SetParent`, `SetWindowPos`).
5. Listening to asynchronous OS events (`WM_DISPLAYCHANGE`, `TaskbarCreated`, power broadcasts, session locks).

Direct invocation of raw C-style Win32 APIs across application codebases introduces catastrophic reliability hazards: untracked raw pointer dereferences, resource handle leaks, undefined behavior from improper struct alignment or uninitialized memory, thread queue synchronization deadlocks, and silent data loss from unhandled error codes.

This specification formalizes the strict separation between portable Rust domain logic and low-level Windows APIs, establishes a zero-cost unsafe encapsulation model, defines RAII ownership semantics for every native resource, and institutes an unyielding error propagation contract.

---

## 2. Core Win32 FFI Safety Principles (Principles A through L)

The native Windows integration layer in KinetiWall is strictly governed by twelve engineering principles:

* **Principle A: Strict Layered Isolation**  
  Application and domain logic (configuration, playlist management, player orchestration, IPC dispatch) must never invoke raw Win32 APIs, import Windows types (`HWND`, `HANDLE`, `WPARAM`), or use `unsafe` blocks directly. All OS interactions must pass through the typed, safe Platform Abstraction Layer.
* **Principle B: Minimal & Audited Unsafe Surface**  
  Unsafe code is confined to the narrowest possible boundary inside the native wrapper modules. Every `unsafe` block must be accompanied by a mandatory `// SAFETY:` invariant comment documenting pointer validity, memory layout, ownership transfer, and calling preconditions.
* **Principle C: RAII Ownership for All Kernel & Win32 Resources**  
  Every native OS resource (`HANDLE`, `HMONITOR`, `HHOOK`, `HPOWERNOTIFY`, `HWINEVENTHOOK`) must be encapsulated within a dedicated Rust struct implementing deterministic `Drop` semantics. Raw integer or pointer handles are never exposed across module boundaries.
* **Principle D: Clear Demarcation of Owned vs Borrowed Handles**  
  The system strictly distinguishes between *owned kernel handles* (which require explicit destruction via `CloseHandle`), *window identifiers* (`HWND`, which represent OS window state managed through windowing lifecycles), and *borrowed references* (`HMONITOR`). Calling `CloseHandle` on an `HWND` or `HMONITOR` is an architectural violation.
* **Principle E: Authoritative Binding Selection (`windows-rs`)**  
  Per `DEP-01` (`docs/dependency_and_licensing/DEPENDENCY_INVENTORY.md`), KinetiWall utilizes Microsoft's official `windows` crate (v0.58) as its authoritative Win32 API binding, supplemented by `windows-sys` where minimal FFI overhead is strictly required. Deprecated crates like `winapi` are strictly forbidden.
* **Principle F: Dual-Key Handle & Identity Validation**  
  Before executing any consequential Win32 operation on an externally discovered handle (such as `SetParent` or `SetWindowPos` on `WorkerW`), the handle must be revalidated via `IsWindow`, class name verification (`RealGetWindowClassW`), process ID verification, and session ID matching to defeat PID/HWND recycling.
* **Principle G: Immediate & Preserved Error Capture**  
  Win32 APIs that report failure via return code and store context in thread-local storage (`GetLastError`) must have their error code captured immediately following the call before any logging, memory allocation, or helper function can overwrite the thread-local error state.
* **Principle H: Zero Panic Across Foreign FFI Boundaries**  
  No native callback (e.g. `WNDPROC`, `MONITORENUMPROC`, `WINEVENTPROC`) may ever unwind or panic across the foreign ABI boundary (`extern "system"`). All callbacks must be wrapped in `std::panic::catch_unwind`, logging errors and returning safe sentinel values (`0` or `FALSE`).
* **Principle I: Thread-Affinity Enforcement**  
  Win32 window creation, message pumps, and `SetParent` calls are inherently thread-affine. The architecture prevents race conditions by binding window operations to dedicated threads and forbidding arbitrary `Send` or `Sync` implementations on thread-affine handles.
* **Principle J: UTF-16 String & Buffer Memory Safety**  
  Conversions between Rust UTF-8 strings and Win32 UTF-16 (`PCWSTR`, `PWSTR`) must never perform unbounded memory allocations, ignore embedded null bytes, or allow buffer overruns. Output buffers must validate lengths and prevent integer overflow.
* **Principle K: Isolation of Undocumented Shell Internals**  
  All operations relying on undocumented Windows Explorer internals (specifically `Progman`, `WorkerW`, and message `0x052C`) must be quarantined behind a dedicated `ShellIntegration` trait, completely decoupled from standard windowing wrappers.
* **Principle L: Testability via Platform Abstraction Traits**  
  All native OS capabilities (Display queries, Window discovery, Process supervision) must be exposed to higher-level controllers through mockable Rust traits, allowing comprehensive unit and fault-injection testing without requiring a live multi-monitor Windows desktop.

---

## 3. Documentation Suite Index

| Deliverable | File Path | Focus & Architectural Scope |
| :--- | :--- | :--- |
| **Index & Principles** | [README.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/native_api_abstraction/README.md) | Architectural index, 12 core FFI safety principles, and requirements traceability matrix. |
| **Deliverable A** | [WINDOWS_API_ABSTRACTION_ARCHITECTURE.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/native_api_abstraction/WINDOWS_API_ABSTRACTION_ARCHITECTURE.md) | 5-tier architecture, domain API inventory, responsibility mapping, capability detection, and undocumented shell isolation. |
| **Deliverable B** | [RUST_FFI_SAFETY_POLICY.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/native_api_abstraction/RUST_FFI_SAFETY_POLICY.md) | Unsafe code boundaries, mandatory safety comments, pointer/buffer rules, UTF-16 conversion, struct layout, thread affinity, and panic barriers. |
| **Deliverable C** | [NATIVE_RESOURCE_OWNERSHIP_MATRIX.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/native_api_abstraction/NATIVE_RESOURCE_OWNERSHIP_MATRIX.md) | Handle typology, owned vs borrowed handles, RAII Drop wrappers, validity constraints, thread affinity, and release order. |
| **Deliverable D** | [WINDOWS_API_ERROR_MAPPING.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/native_api_abstraction/WINDOWS_API_ERROR_MAPPING.md) | Return-value taxonomy, `GetLastError` capture rules, Win32-to-Rust error mapping, recovery actions, and diagnostic logging schema. |
| **Deliverable E** | [WINDOWS_API_ABSTRACTION_TEST_PLAN.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/native_api_abstraction/WINDOWS_API_ABSTRACTION_TEST_PLAN.md) | Mockable trait design, unit tests, Windows integration tests, unsafe-code review checklist, and performance benchmarks (`FFI-001..012`). |
| **Deliverable F** | [WINDOWS_API_COMPATIBILITY_DECISIONS.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/native_api_abstraction/WINDOWS_API_COMPATIBILITY_DECISIONS.md) | Platform capability matrix, Decision Register (`DEC-FFI-01..08`), Risk Register (`RSK-FFI-01..08`), and open review items. |

---

## 4. Requirements Traceability Matrix

The native Win32 abstraction architecture satisfies and enforces the following requirements and decisions:

| Requirement / Decision | Description | Governing Specification Section |
| :--- | :--- | :--- |
| **`KW-F-001`** | Core engine initialization and native desktop shell integration | `WINDOWS_API_ABSTRACTION_ARCHITECTURE.md` §3 |
| **`KW-F-007`** | Multi-monitor independent display configuration & geometry | `WINDOWS_API_ABSTRACTION_ARCHITECTURE.md` §4 |
| **`KW-F-008`** | Power-state and fullscreen hardware occlusion detection | `NATIVE_RESOURCE_OWNERSHIP_MATRIX.md` §3 |
| **`KW-NF-001`** | CPU utilization limits ($\le 1.0\%$ idle background overhead) | `WINDOWS_API_ABSTRACTION_TEST_PLAN.md` §4 |
| **`KW-NF-006`** | Graceful shutdown and rapid exit response ($\le 250\text{ ms}$) | `NATIVE_RESOURCE_OWNERSHIP_MATRIX.md` §4 |
| **`ADR-0002`** | Isolated multi-process topology (Core, UI, Per-Display Players) | `WINDOWS_API_ABSTRACTION_ARCHITECTURE.md` §2 |
| **`ADR-0003`** | Async runtime & event loop selection (Tokio + Named Pipes) | `WINDOWS_API_ABSTRACTION_ARCHITECTURE.md` §3 |
| **`DEC-PRF-08`** | Windows Job Object orphan process containment | `NATIVE_RESOURCE_OWNERSHIP_MATRIX.md` §2 |
| **`DEC-SEC-01`** | Defense-in-depth security model & memory safety enforcement | `RUST_FFI_SAFETY_POLICY.md` §2 |
| **`DEC-DSK-01..08`**| Windows desktop integration, WorkerW, and SetParent rules | `WINDOWS_API_ABSTRACTION_ARCHITECTURE.md` §5 |
| **`DEC-FFI-01..08`**| Dedicated FFI abstraction, RAII, and error-handling decisions | `WINDOWS_API_COMPATIBILITY_DECISIONS.md` §2 |

