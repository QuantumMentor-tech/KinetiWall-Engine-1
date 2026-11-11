# Resource Ownership & Reclamation Matrix
**Document ID:** `KW-LFC-003`  
**Deliverable:** C (Resource Ownership Matrix)  
**Parent Document:** [README.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/lifecycle_and_shutdown/README.md)  
**Classification:** Core System Architecture / Operational Reliability Standard  

---

## 1. Executive Summary & Ownership Rules

In KinetiWall, resource management follows strict RAII (Resource Acquisition Is Initialization) semantics coupled with deterministic Windows kernel resource reclamation patterns. Every allocated resource has:
1. Exactly one authoritative owner process and subsystem.
2. A deterministic allocation trigger and strictly defined lifetime.
3. An explicit release trigger and mandatory destruction sequence.
4. A completion verification signal and maximum release timeout.
5. An automated leak detection method to catch regressions during QA testing.

---

## 2. Master Resource Allocation & Reclamation Matrix

The following table comprehensively catalogs every internal and OS resource across the Core supervisor, User Interface, and Player subprocesses:

| Resource Identifier | Resource Category | Owning Process & Subsystem | Creation Trigger | Lifetime Scope | Release Trigger | Destruction Order | Completion Signal | Timeout Budget | Leak Detection Method |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **`RES-01: JobObject`** | OS Kernel Handle | `kinetiwall-core` (Supervisor) | Core Initialization | Process Lifetime | Core Teardown (Stage 14) | 14 (Late) | `CloseHandle()` returns nonzero | $10\text{ ms}$ | Kernel object count check (`ntdll!NtQuerySystemInformation`) |
| **`RES-02: SingleInstanceMutex`** | Win32 Sync Object | `kinetiwall-core` (Main) | Core Startup Entry | Process Lifetime | Core Teardown (Stage 15) | 15 (Final) | `ReleaseMutex()` & `CloseHandle()` | $5\text{ ms}$ | Win32 `OpenMutexW` verification post-exit |
| **`RES-03: CorePipeServer`** | OS Named Pipe | `kinetiwall-core` (IPC Server) | IPC Module Startup | Process Lifetime | Core Teardown (Stage 15) | 15 (Final) | `DisconnectNamedPipe()` & `CloseHandle()` | $10\text{ ms}$ | Handle table inspection via WinDbg/Process Explorer |
| **`RES-04: PlayerProcessHandle`** | OS Process Handle | `kinetiwall-core` (Supervisor) | `CreateProcessW` | Player Lifetime | Player Exit / Terminate | 10 (Post-Exit) | `CloseHandle(hProcess)` | $5\text{ ms}$ | Core handle count tracking in telemetry |
| **`RES-05: TrayNotificationIcon`** | Shell Shell_NotifyIcon | `kinetiwall-core` (Tray Host) | UI Subsystem Start | Indefinite / App Life | Engine Exit (Stage 13) | 13 | `Shell_NotifyIconW(NIM_DELETE)` | $10\text{ ms}$ | Automated desktop shell screenshot & tray audit |
| **`RES-06: PowerEventSubscription`** | OS Power Broadcast | `kinetiwall-core` (Power Monitor) | Power Monitor Init | Indefinite / App Life | Engine Exit (Stage 12) | 12 | `UnregisterPowerSettingNotification()` | $10\text{ ms}$ | Power telemetry listener check |
| **`RES-07: SessionEventSubscription`** | OS Terminal Services | `kinetiwall-core` (Session Monitor)| Session Monitor Init| Indefinite / App Life | Engine Exit (Stage 12) | 12 | `WTSUnRegisterSessionNotification()` | $10\text{ ms}$ | Terminal Services session query |
| **`RES-08: WinEventHook (Display)`** | Win32 Event Hook | `kinetiwall-core` (Display Monitor)| Display Monitor Init| Indefinite / App Life | Engine Exit (Stage 12) | 12 | `UnhookWinEvent()` returns `TRUE` | $5\text{ ms}$ | Global hook inspection via Process Hacker |
| **`RES-09: SQLiteConnectionPool`** | Database Handles | `kinetiwall-core` (Storage Engine)| DB Init | Indefinite / App Life | Engine Exit (Stage 11) | 11 | `sqlite3_close_v2()` returns `SQLITE_OK`| $50\text{ ms}$ | File lock check on `.db-wal` and `.db-shm` |
| **`RES-10: PlayerPipeClient`** | OS Named Pipe | `kinetiwall-player-*` (IPC Client) | Player Process Init | Player Lifetime | Player Teardown (Stage 8) | 8 | `FlushFileBuffers()` & `CloseHandle()` | $15\text{ ms}$ | Pipe disconnection event emitted on Core |
| **`RES-11: PlayerWindowHWND`** | Win32 Window | `kinetiwall-player-*` (Window Host)| Player Window Init | Player Lifetime | Player Teardown (Stage 5) | 5 | `DestroyWindow(hWnd)` returns `TRUE` | $10\text{ ms}$ | Desktop child window enumeration (`EnumChildWindows`)|
| **`RES-12: D3D11SwapChain`** | DirectX 11 / DXGI | `kinetiwall-player-*` (Renderer) | Renderer Init | Player Lifetime | Player Teardown (Stage 6) | 6 | `IDXGISwapChain::Release() == 0` | $20\text{ ms}$ | DXGI Leak Tracking (`D3D11_CREATE_DEVICE_DEBUG`) |
| **`RES-13: D3D11DeviceContext`**| DirectX 11 API | `kinetiwall-player-*` (Renderer) | Renderer Init | Player Lifetime | Player Teardown (Stage 6) | 6 | `ID3D11DeviceContext::ClearState()`, Release| $10\text{ ms}$ | DXGI Debug Object count |
| **`RES-14: mpvCoreInstance`** | Media Decoder API | `kinetiwall-player-video` (Engine) | Video Engine Init | Wallpaper Lifetime| Wallpaper Switch / Stop (Stage 6)| 6 | `mpv_terminate_destroy()` returns | $50\text{ ms}$ | Worker thread enumeration in player process |
| **`RES-15: WebView2Controller`**| Chromium COM Object | `kinetiwall-player-web` (Engine) | Web Engine Init | Wallpaper Lifetime| Wallpaper Switch / Stop (Stage 6)| 6 | `ICoreWebView2Controller::Close()` | $75\text{ ms}$ | Edge WebView2 child process count check |
| **`RES-16: WASAPIAudioClient`** | Core Audio COM API | `kinetiwall-player-*` (Audio Subsys)| Audio Stream Init | Wallpaper Lifetime| Wallpaper Stop (Stage 7) | 7 | `IAudioClient::Stop()`, Release | $15\text{ ms}$ | Windows Volume Mixer active stream check |
| **`RES-17: ConfigFileTempHandle`** | OS File Handle | `kinetiwall-core` (Config Engine) | Config Flush Trigger| Transient (< 10 ms)| Post-Write Flush | Immediate | `CloseHandle(hTempFile)` | $5\text{ ms}$ | Open file handles on `config.json.tmp.*` |
| **`RES-18: UIWindowHWND`** | Win32 / Slint Window | `kinetiwall-ui` (GUI Shell) | UI Launch Trigger | Ephemeral (User Session)| User Close / Core Quit (Stage 3) | 3 | `DestroyWindow(hWnd)`, `ExitProcess()` | $20\text{ ms}$ | Desktop top-level window check |

---

## 3. Subsystem-Specific Teardown Protocols

### 3.1 mpv Video Decoder Teardown Protocol (`RES-14`)

Video playback uses an embedded `mpv` rendering context linked to Direct3D 11. To prevent hard deadlocks where `mpv_terminate_destroy()` hangs waiting for audio hardware buffers or GPU presentation:

```
[Stop Trigger Issued]
         |
         v
1. Unhook Audio Output: mpv_set_property_string(ctx, "ao", "null")
         |
         v
2. Issue Asynchronous Stop: mpv_command_string(ctx, "stop")
         |
         v
3. Detach Render Context: mpv_render_context_free(render_ctx)
         |
         v
4. Terminate Core: mpv_terminate_destroy(ctx)
         |
         v
5. Verify Thread Pool Teardown (Timeout 50ms)
```

If `mpv_terminate_destroy()` does not return within $50\text{ ms}$, the player process marks the subsystem timed-out and immediately invokes `ExitProcess(EXIT_CODE_SUBPROCESS_DECODER_HANG = 0xFD)`, bypassing further cooperative cleanup and allowing the OS kernel to release locked memory.

### 3.2 Microsoft Edge WebView2 Teardown Protocol (`RES-15`)

Interactive web wallpapers spawn Chromium utility processes (`msedgewebview2.exe`). Teardown must cleanly terminate both the host COM controller and child Chromium processes:

1. **Host Navigation Freeze:** WebView2 controller executes `ICoreWebView2::Navigate(L"about:blank")` to instantly release DOM trees, JavaScript timers, and GPU webGL contexts.
2. **Controller Close:** Host calls `ICoreWebView2Controller::Close()`. This explicitly signals child `msedgewebview2.exe` helper processes to shut down.
3. **Environment Teardown:** Host releases the `ICoreWebView2Environment` interface.
4. **Child Process Verification:** Core verifies that child Chromium processes associated with the web player's Job Object have terminated.

### 3.3 WASAPI Audio Teardown Protocol (`RES-16`)

Audio streaming utilizes Windows WASAPI in shared mode. Teardown must avoid audio pops, clicks, or orphaned audio sessions in the Windows Volume Mixer:

1. **Volume Ramp Down:** Fast linear volume attenuation ($5\text{ ms}$ duration) to 0.0 amplitude.
2. **Stop Stream:** Call `IAudioClient::Stop()`.
3. **Reset Audio Buffer:** Call `IAudioClient::Reset()`.
4. **Release Interfaces:** Release `IAudioRenderClient`, `IAudioClient`, and `IMMDevice`.

---

## 4. Automated Leak Detection & Verification Methodology

During integration testing, automated tests validate zero resource leaks using the following test assertions:

1. **GDI & USER Object Auditing:** Win32 API `GetGuiResources(GetCurrentProcess(), GR_GDIOBJECTS)` and `GetGuiResources(GetCurrentProcess(), GR_USEROBJECTS)` are measured before launch and after shutdown. Any Delta $> 0$ fails the test suite.
2. **Kernel Handle Count Auditing:** `GetProcessHandleCount()` is polled. Post-shutdown handle counts must return to baseline ($0$ for terminated subprocesses, baseline $\pm 2$ for Core during idle).
3. **DirectX Object Validation:** D3D11 debug layer (`D3D11_CREATE_DEVICE_DEBUG`) is enabled in test configurations. Calling `ReportLiveDeviceObjects(D3D11_RLDO_DETAIL)` upon exit must report 0 live objects. Any live texture or swapchain triggers a catastrophic test failure.
4. **Desktop Window Hierarchy Sweep:** Automated tests enumerate all child windows of `Progman` and `WorkerW` using `EnumChildWindows`. Following shutdown, no windows belonging to `kinetiwall-player-*.exe` or `kinetiwall-ui.exe` may remain attached.

