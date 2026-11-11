# KINETIWALL — IPC CONTRACTS, SECURITY THREAT MODEL & TRUST BOUNDARIES
## Document ID: KW-SEC-000 | Phase 2: System Architecture & Technical Decisions

---

### Executive Summary

The **KinetiWall IPC and Security Architecture Specification** establishes the authoritative security foundation for inter-process communication (IPC), command authorization, untrusted content containment, threat modeling, and fault recovery across the KinetiWall live wallpaper engine.

KinetiWall operates as a decoupled, multi-process desktop system on Windows 10 and Windows 11. To prevent privilege escalation, malicious native execution from untrusted wallpapers, unauthorized process control, and cross-session pipe hijacking, this specification defines a **Zero-Trust Local IPC Architecture**:
1. **Windows Named Pipes with Explicit Security Descriptors (SDDL)**: IPC endpoints reject default unauthenticated pipe security. Endpoints are strictly scoped to the local Windows logon session, enforcing explicit discretionary access control lists (DACLs) restricted to the current user's security identifier (SID).
2. **Defensive Client Identification & Impersonation Prevention**: Named Pipe servers utilize `GetNamedPipeClientProcessId` and client connection tokens configured with `SECURITY_IDENTIFICATION` to permanently neutralize Win32 pipe impersonation risks (`ImpersonateNamedPipeClient` abuse).
3. **Rigid Separation of Privileged Core and Untrusted Players**: The Controller Core (`kinetiwall-core.exe`) is the sole entity authorized to touch the Windows desktop tree, write application state, or supervise processes. The Gallery UI (`kinetiwall-ui.exe`) and Player subprocesses (`kinetiwall-player-video.exe`, `kinetiwall-player-web.exe`) hold zero direct administrative capabilities.
4. **Hermetic Web Wallpaper Sandboxing**: Interactive HTML5/WebGL wallpapers run within Chromium/Edge WebView2 isolated sandboxes. Local filesystem access, native Win32 APIs, shell execution, unrestricted networking, and raw audio capture are completely blocked. Web wallpapers communicate solely via a constrained JSON postMessage bridge with downsampled, derived audio frequency buckets (zero raw PCM audio).
5. **Deterministic Threat Model & STRIDE Register**: 20 formal threat definitions (`KW-T-001` through `KW-T-020`) systematically evaluate spoofing, tampering, repudiation, information disclosure, denial of service, and elevation of privilege across all system attack surfaces.

---

### Document Navigation Index

| Document ID | File Name | Scope & Key Contents |
| :--- | :--- | :--- |
| **KW-SEC-000** | [README.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/security_and_ipc/README.md) | Architectural index, core security principles, navigation, and requirements traceability matrix. |
| **KW-SEC-001** | [IPC_ARCHITECTURE_AND_ENDPOINTS.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/security_and_ipc/IPC_ARCHITECTURE_AND_ENDPOINTS.md) | Transport evaluation, Named Pipe endpoint topology, session-scoped naming, Win32 DACL security descriptors, client PID verification, impersonation defense. |
| **KW-SEC-002** | [IPC_MESSAGE_CONTRACTS.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/security_and_ipc/IPC_MESSAGE_CONTRACTS.md) | Exhaustive message catalog, envelope schema, correlation IDs, typed payloads for state, assignments, playback, lifecycle, policy, and audio visualization. |
| **KW-SEC-003** | [AUTHORIZATION_AND_TRUST_BOUNDARIES.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/security_and_ipc/AUTHORIZATION_AND_TRUST_BOUNDARIES.md) | Trust boundary definitions, Mermaid trust boundary and data-flow diagrams, comprehensive Command Authorization Matrix (Principals $\times$ Operations). |
| **KW-SEC-004** | [SECURITY_THREAT_MODEL.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/security_and_ipc/SECURITY_THREAT_MODEL.md) | STRIDE methodology, protected asset inventory, threat actors, attack surfaces, formal threat records (`KW-T-001`..`020`), risk scores, mitigations. |
| **KW-SEC-005** | [VALIDATION_AND_FAILURE_SEMANTICS.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/security_and_ipc/VALIDATION_AND_FAILURE_SEMANTICS.md) | Input validation bounds, numerical limits, request lifecycles, timeout rules, idempotency, retry policies, race-condition reconciliation, crash recovery. |
| **KW-SEC-006** | [WEBVIEW2_SECURITY_SPECIFICATION.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/security_and_ipc/WEBVIEW2_SECURITY_SPECIFICATION.md) | WebView2 security boundaries, virtual host mapping (`https://kinetiwall.local/`), CSP headers, restricted postMessage API, audio spectrum downsampling. |
| **KW-SEC-007** | [SECURITY_TEST_CATALOG_AND_DECISIONS.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/security_and_ipc/SECURITY_TEST_CATALOG_AND_DECISIONS.md) | Security Acceptance Test Catalog (`SAT-001`..`020`, marked `NOT RUN`), Security Decision Register (`DEC-SEC-01`..`08`), and provisional assumptions. |

---

### Core Security Architecture Principles

1. **Principle of Least Privilege**:
   Components operate with the absolute minimum privileges required to perform their functions. Wallpaper players cannot create processes, edit global configuration, or inspect other monitors. Untrusted web wallpapers execute inside restricted renderer processes with zero native Win32 bindings.
2. **Defensive Non-Implicit Trust (Zero-Trust Local IPC)**:
   The Controller Core never trusts a command simply because it arrived over a local pipe. Every incoming message envelope is authenticated against the connection's validated OS process identifier, verified against the Command Authorization Matrix, and validated against strict schema and payload bounds.
3. **Impersonation Immunity**:
   All Named Pipe connections explicitly disallow token impersonation by specifying `SECURITY_IDENTIFICATION` on client handles. Even if a malicious client connects to an engine pipe or an engine client connects to a spoofed pipe, the receiver cannot impersonate the caller's Windows token.
4. **Hermetic Session Isolation**:
   IPC pipe names include the Windows Logon Session ID (`GetTokenInformation(TokenSessionId)`). Multi-user environments (e.g. Fast User Switching, Terminal Services) cannot see, connect to, or tamper with KinetiWall endpoints belonging to another logged-in user.
5. **Fail-Closed and Fail-Safe Posture**:
   When a message fails validation, a protocol version mismatch is detected, or an unauthorized action is attempted, the connection is terminated or the request is rejected with a structured error. State mutations are rolled back; the engine never "fails open" to accommodate malformed inputs.
6. **Local Privacy Guarantee**:
   Zero personal data, wallpaper usage history, crash minidumps, or audio streams leave the local machine. Audio visualization never captures, records, or exposes raw PCM audio samples; it computes downsampled frequency spectrums in-memory and discards the raw audio immediately.

---

### Requirements Traceability Matrix

| Requirement ID | Requirement Scope | IPC & Security Specification Document |
| :--- | :--- | :--- |
| **KW-F-016** | Web Wallpaper Execution & Sandboxing | [WEBVIEW2_SECURITY_SPECIFICATION.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/security_and_ipc/WEBVIEW2_SECURITY_SPECIFICATION.md) §1-§4 |
| **KW-F-018** | Dynamic Property Synchronization | [IPC_MESSAGE_CONTRACTS.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/security_and_ipc/IPC_MESSAGE_CONTRACTS.md) §6, [VALIDATION_AND_FAILURE_SEMANTICS.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/security_and_ipc/VALIDATION_AND_FAILURE_SEMANTICS.md) §1 |
| **KW-F-020** | Multi-Monitor Topology & Assignments | [IPC_MESSAGE_CONTRACTS.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/security_and_ipc/IPC_MESSAGE_CONTRACTS.md) §4 |
| **KW-F-024** | Audio Visualization Pipeline & Privacy | [WEBVIEW2_SECURITY_SPECIFICATION.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/security_and_ipc/WEBVIEW2_SECURITY_SPECIFICATION.md) §5 |
| **KW-NF-012** | Process Crash Isolation & Resilience | [VALIDATION_AND_FAILURE_SEMANTICS.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/security_and_ipc/VALIDATION_AND_FAILURE_SEMANTICS.md) §5 |
| **KW-NF-018** | Sandboxed Execution of Untrusted Content | [AUTHORIZATION_AND_TRUST_BOUNDARIES.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/security_and_ipc/AUTHORIZATION_AND_TRUST_BOUNDARIES.md) §1-§3 |
| **ADR-0002** | Multi-Process Architecture | [AUTHORIZATION_AND_TRUST_BOUNDARIES.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/security_and_ipc/AUTHORIZATION_AND_TRUST_BOUNDARIES.md) §1 |
| **ADR-0003** | Local IPC via Windows Named Pipes | [IPC_ARCHITECTURE_AND_ENDPOINTS.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/security_and_ipc/IPC_ARCHITECTURE_AND_ENDPOINTS.md) §1-§4 |
| **ADR-0008** | WebView2 Virtual Host Mapping Sandboxing | [WEBVIEW2_SECURITY_SPECIFICATION.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/security_and_ipc/WEBVIEW2_SECURITY_SPECIFICATION.md) §2 |
| **CTR-01..03**| IPC Contracts (Core-UI, Core-Player) | [IPC_MESSAGE_CONTRACTS.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/security_and_ipc/IPC_MESSAGE_CONTRACTS.md) §1-§7 |

