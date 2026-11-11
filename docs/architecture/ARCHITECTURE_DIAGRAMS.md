# KinetiWall — System Architecture Diagrams
**Document ID:** `KW-ARC-002`  
**Deliverable:** Deliverable B — Architecture Diagrams  
**Phase:** 2 — System Architecture & Technical Decisions  
**Step:** 7 of 90 — System Architecture & Architecture Decision Records  
**Status:** COMPLETE (Auditable Mermaid Visual Models)  
**Authority:** Principal Software Architect & Rust Systems Lead  
**Date:** 2026-11-10  

---

## 1. System Context Diagram (C4 Level 1)

Visualizes KinetiWall in relation to the Windows operating system, display hardware, local filesystem, and user interactions.

```mermaid
graph TD
    USER["User / Desktop Operator"]
    
    subgraph OS["Windows 11 / 10 Operating System"]
        EXPLORER["Windows Explorer\n(Progman / WorkerW / Taskbar)"]
        DWM["Desktop Window Manager (DWM)"]
        AUDIO["Windows Core Audio (WASAPI)"]
        POWER["Windows Power Management"]
        FS["Local AppData Storage\n(%LOCALAPPDATA%\\KinetiWall)"]
    end
    
    subgraph APP["KinetiWall Architecture Boundary"]
        CORE["kinetiwall-core.exe\n(Pure Rust Daemon, 8-12 MB)"]
        UI["kinetiwall-ui.exe\n(Tauri v2 Ephemeral Gallery, 0 MB Closed)"]
        VPLAY["kinetiwall-player-video.exe\n(libmpv Direct3D 11VA)"]
        WPLAY["kinetiwall-player-web.exe\n(WebView2 Sandboxed WebGL)"]
    end
    
    USER -->|Clicks Tray / Customizes| UI
    USER -->|Clicks Desktop Icons / Focuses Apps| EXPLORER
    
    UI <-->|Named Pipe IPC JSON-RPC| CORE
    CORE -->|Job Object Supervision| VPLAY
    CORE -->|Job Object Supervision| WPLAY
    CORE <-->|Named Pipe IPC Controls| VPLAY
    CORE <-->|Named Pipe IPC Controls| WPLAY
    
    CORE -->|RegisterWindowMessage / 0x052C| EXPLORER
    CORE -->|IAudioClient Loopback| AUDIO
    CORE -->|RegisterPowerSettingNotification| POWER
    CORE -->|Atomic Read/Write config.json| FS
    
    VPLAY -->|HWND Child Reparenting| EXPLORER
    WPLAY -->|HWND Child Reparenting| EXPLORER
    VPLAY -->|Direct3D 11 Present| DWM
    WPLAY -->|Direct3D 11 Present| DWM

    classDef user fill:#2b2d42,stroke:#8d99ae,stroke-width:2px,color:#edf2f4;
    classDef os fill:#1d3557,stroke:#457b9d,stroke-width:2px,color:#f1faee;
    classDef core fill:#003566,stroke:#ffc300,stroke-width:2px,color:#ffd60a;
    classDef comp fill:#1b4332,stroke:#2d6a4f,stroke-width:2px,color:#d8f3dc;

    class USER user;
    class EXPLORER,DWM,AUDIO,POWER,FS os;
    class CORE,UI core;
    class VPLAY,WPLAY comp;
```

---

## 2. Container & Process Topology Diagram (C4 Level 2)

Details the multi-process layout, IPC communication channels, Windows Job Object containment, and memory budgets.

```mermaid
graph TD
    subgraph ControllerContainer["Process 1: Core Controller Daemon (Session ID >= 1)"]
        DAEMON["kinetiwall-core.exe\n(Pure Rust Background Daemon)\nRAM: 8–12 MB | Idle CPU: <=0.10%"]
        TRAY["Win32 Shell Notify Icon\n(Tray Context Menu & Events)"]
        IPCSERVER["Named Pipe Server\n(\\\\.\\pipe\\kinetiwall-ipc-{SessionId})\nDACL: Local Interactive Logon SID"]
        SUPERVISOR["Process Supervisor\n(Windows Job Object Creator)"]
    end

    subgraph UIContainer["Process 2: On-Demand Gallery UI (Ephemeral)"]
        TAURI["kinetiwall-ui.exe (Tauri v2)\nRAM: 70–120 MB Active | 0 MB Closed\nAction: std::process::exit(0) on close"]
        IPCCLIENT["Named Pipe Client (IPC Caller)"]
        FRONTEND["Web Frontend (Svelte/React + Tailwind)\nWallpaper Grid, Settings, Property Sliders"]
    end

    subgraph JobSandbox["Windows Job Object Container (JOB_OBJECT_LIMIT_KILL_ON_JOB_CLOSE)"]
        subgraph VideoProcess["Process 3: Isolated Video Player"]
            VEXEC["kinetiwall-player-video.exe\nRAM: 45–90 MB | D3D11VA HW Looping"]
            MPV["libmpv-2.dll (LGPL v2.1+ Dynamic Link)\nvo=gpu-next, hwdec=d3d11va"]
        end

        subgraph WebProcess["Process 4: Isolated Web Player"]
            WEXEC["kinetiwall-player-web.exe\nRAM: 120–250 MB | Quota: 500 MB Limit"]
            WEBVIEW["Microsoft Edge WebView2 Evergreen\nVirtual Host: https://kinetiwall.local/"]
        end
    end

    DAEMON --- TRAY
    DAEMON --- IPCSERVER
    DAEMON --- SUPERVISOR

    TAURI --- IPCCLIENT
    TAURI --- FRONTEND

    IPCCLIENT <==>|Authenticated JSON-RPC Messages| IPCSERVER
    SUPERVISOR ==>|Spawns & Binds to Job Object| VEXEC
    SUPERVISOR ==>|Spawns & Binds to Job Object| WEXEC
    
    IPCSERVER <==>|IPC Controls (Play/Pause/Volume)| VEXEC
    IPCSERVER <==>|IPC Controls & Dynamic Props| WEXEC
    
    VEXEC --- MPV
    WEXEC --- WEBVIEW

    classDef daemon fill:#003566,stroke:#ffc300,stroke-width:2px,color:#ffd60a;
    classDef ui fill:#d90429,stroke:#ef233c,stroke-width:2px,color:#ffffff;
    classDef job fill:#14213d,stroke:#fca311,stroke-width:2px,color:#ffffff;
    classDef player fill:#1b4332,stroke:#2d6a4f,stroke-width:2px,color:#d8f3dc;

    class ControllerContainer,DAEMON,TRAY,IPCSERVER,SUPERVISOR daemon;
    class UIContainer,TAURI,IPCCLIENT,FRONTEND ui;
    class JobSandbox job;
    class VideoProcess,VEXEC,MPV,WebProcess,WEXEC,WEBVIEW player;
```

---

## 3. Component Responsibility Diagram (C4 Level 3)

Deconstructs `kinetiwall-core.exe` into its 10 cohesive internal subsystem modules.

```mermaid
graph TD
    subgraph CoreDaemon["kinetiwall-core.exe Subsystem Architecture"]
        ALM["1. AppLifecycleManager\n(Startup, Tray Icon, Shutdown)"]
        PROC["2. ProcessOrchestrator\n(Job Objects, Spawning, Crash Backoff)"]
        IPC["3. IpcServer & Authorizer\n(Named Pipe Server, DACL, Serde JSON)"]
        DHM["4. DesktopHostManager\n(Progman/WorkerW 0x052C, Explorer Watcher)"]
        DTM["5. DisplayTopologyManager\n(EnumDisplayMonitors, Mixed DPI, Span/Clone)"]
        RPE["6. ResourcePolicyEngine\n(SetWinEventHook Occlusion, Fullscreen Pause)"]
        PSM["7. PowerStateManager\n(GetSystemPowerStatus, Battery Saver Rules)"]
        LIB["8. LibraryRepository\n(project.json Schema V1, Zip-Slip Guard)"]
        CONF["9. ConfigurationRepository\n(Atomic config.json.bak Persistence)"]
        AUD["10. AudioCaptureEngine\n(WASAPI Loopback, Pure-Rust SIMD FFT)"]
    end

    ALM --> CONF
    ALM --> DHM
    ALM --> DTM
    ALM --> IPC
    
    IPC <--> ALM
    IPC <--> LIB
    IPC <--> PROC
    
    PROC --> DHM
    PROC --> DTM
    
    RPE -->|Pause / Resume Triggers| PROC
    PSM -->|Battery Transition Events| RPE
    DHM -->|Shell Attachment HWNDs| PROC
    AUD -->|Frequency Array IPC Stream| PROC

    classDef core fill:#003566,stroke:#ffc300,stroke-width:2px,color:#ffd60a;
    class ALM,PROC,IPC,DHM,DTM,RPE,PSM,LIB,CONF,AUD core;
```

---

## 4. IPC Architecture & Trust Boundaries Diagram

Illustrates security zoning, authorization gates, and data isolation between trusted and untrusted components.

```mermaid
graph LR
    subgraph UntrustedZone["Untrusted Security Zone"]
        JS["Web Wallpaper JavaScript\n(HTML5 / WebGL / Canvas)"]
        PKG["Imported Wallpaper Packages\n(.zip / .kwall / media files)"]
    end

    subgraph SandboxedZone["Restricted Sandbox Zone (Medium IL)"]
        WEBPLAYER["kinetiwall-player-web.exe\n(WebView2 Virtual Host Sandbox)\nCORS: Blocked | Cameras/Mics: Denied"]
        VIDEOPLAYER["kinetiwall-player-video.exe\n(libmpv D3D11VA Host)"]
    end

    subgraph TrustedZone["Trusted User Space Zone (Interactive Logon SID)"]
        CORE["kinetiwall-core.exe\n(Named Pipe IPC Server)\nDACL: Interactive Logon SID Only"]
        UI["kinetiwall-ui.exe (Tauri v2)\n(Authenticated Admin Client)"]
        CONF["config.json & Wallpapers Disk Storage"]
    end

    JS -.->|window.chrome.webview.postMessage| WEBPLAYER
    PKG -.->|Import Stream| UI
    
    UI ===|Named Pipe: ApplyWallpaper, SetConfig| CORE
    WEBPLAYER ===|Named Pipe: StatusReady, ErrorNotify| CORE
    VIDEOPLAYER ===|Named Pipe: PlaybackAck, EOF| CORE
    
    CORE ===|Job Object Hard RAM/Process Limits| WEBPLAYER
    CORE ===|Job Object Hard RAM/Process Limits| VIDEOPLAYER
    CORE ===|Atomic Write| CONF

    classDef untrusted fill:#6b705c,stroke:#a5a58d,stroke-width:2px,color:#ffffff;
    classDef sandboxed fill:#cb997e,stroke:#ddbea9,stroke-width:2px,color:#ffffff;
    classDef trusted fill:#003566,stroke:#ffc300,stroke-width:2px,color:#ffd60a;

    class UntrustedZone,JS,PKG untrusted;
    class SandboxedZone,WEBPLAYER,VIDEOPLAYER sandboxed;
    class TrustedZone,CORE,UI,CONF trusted;
```

---

## 5. Sequence Diagram 1: Startup & Wallpaper Presentation

```mermaid
sequenceDiagram
    autonumber
    actor User as User / Windows Session
    participant Core as kinetiwall-core.exe
    participant Shell as Windows Shell (Progman/WorkerW)
    participant Conf as config.json
    participant Player as kinetiwall-player-video.exe
    participant Screen as Desktop Surface

    User->>Core: Launch Session Daemon (Logon / Startup)
    Core->>Conf: Load & Validate config.json
    Conf-->>Core: Active Profile: "Cyberpunk_Night.mp4" (Monitor 1)
    Core->>Shell: SendMessageTimeout(Progman, 0x052C)
    Shell-->>Core: Spawned WorkerW HWND
    Core->>Core: DisplayTopologyManager::EnumerateMonitors()
    Core->>Player: CreateProcess() bound to Job Object
    Player->>Player: Initialize D3D11 Device & libmpv
    Player->>Core: IPC Handshake: PlayerReady(PlayerHWND)
    Core->>Shell: SetParent(PlayerHWND, WorkerW)
    Core->>Shell: SetWindowPos(PlayerHWND, HWND_BOTTOM, Mon1Rect)
    Core->>Player: IPC Command: Play("Cyberpunk_Night.mp4", loop=inf)
    Player->>Screen: Present D3D11 Swapchain Backbuffer
    Screen-->>User: Live Wallpaper visible behind desktop icons
```

---

## 6. Sequence Diagram 2: Ephemeral Gallery UI Lifecycle

```mermaid
sequenceDiagram
    autonumber
    actor User as User
    participant Tray as Win32 System Tray
    participant Core as kinetiwall-core.exe
    participant UI as kinetiwall-ui.exe (Tauri)
    participant OS as Windows Task Manager

    User->>Tray: Double-Click Tray Icon
    Tray->>Core: WM_COMMAND(ID_OPEN_GALLERY)
    Core->>OS: Spawn kinetiwall-ui.exe
    OS->>UI: Process Initialized (70–120 MB RAM allocated)
    UI->>Core: Connect Named Pipe (\\\\.\\pipe\\kinetiwall-ipc-{SessionId})
    Core-->>UI: IPC Response: ActiveMonitors, LibraryCatalog, ConfigState
    UI->>User: Render Gallery Grid & Active Wallpaper Inspector
    User->>UI: Click 'X' (Close Window)
    UI->>UI: Intercept WindowClose Event
    UI->>Core: IPC Notify: GalleryClosing()
    UI->>OS: std::process::exit(0)
    OS-->>User: Process vanishes from Task Manager (0 MB RAM resident)
    Note over Core: Active wallpapers continue playing uninterrupted (8–12 MB Core)
```

---

## 7. Sequence Diagram 3: Explorer Crash & Automated Re-Attachment

```mermaid
sequenceDiagram
    autonumber
    actor OS as Windows OS / Crash Event
    participant Shell as explorer.exe (Restarting)
    participant Core as kinetiwall-core.exe
    participant Player as kinetiwall-player-video.exe

    OS->>Shell: Terminate explorer.exe (Taskkill / Crash)
    Note over Core,Player: Desktop host HWND invalidated
    Core->>Player: IPC Command: PauseRendering()
    OS->>Shell: Launch replacement explorer.exe
    Shell->>OS: Broadcast Registered Message: "TaskbarCreated"
    Core->>Core: OnMessage("TaskbarCreated") Received
    Core->>Shell: Broadcast 0x052C to new Progman
    Shell-->>Core: New WorkerW HWND Created
    Core->>Shell: Re-Parent PlayerHWND beneath new WorkerW
    Core->>Shell: SetWindowPos(PlayerHWND, HWND_BOTTOM, MonRect)
    Core->>Player: IPC Command: ResumeRendering()
    Player-->>Shell: Swapchain presenting behind icons (Recovery in <=2.5s)
```

---

## 8. Sequence Diagram 4: Fullscreen Game Occlusion & Instant Pause

```mermaid
sequenceDiagram
    autonumber
    actor User as User
    participant Game as 3D Game (e.g. Cyberpunk 2077)
    participant WinHook as Win32 SetWinEventHook
    participant Core as kinetiwall-core.exe
    participant Player as kinetiwall-player-video.exe

    User->>Game: Focus Fullscreen 3D Game
    Game->>WinHook: EVENT_SYSTEM_FOREGROUND (GameHWND)
    WinHook->>Core: Callback(EVENT_SYSTEM_FOREGROUND, GameHWND)
    Core->>Core: ResourcePolicyEngine::EvaluateOcclusion(GameHWND)
    Note over Core: GameHWND rect covers Display 1 (Fullscreen detected)
    Core->>Player: IPC Command: Pause(Immediate, FreezeFrame=true)
    Player->>Player: Halt D3D11 Present Loop; Freeze Swapchain
    Note over Player: Player GPU load drops to 0.00% in <=50ms
    User->>Game: Plays game with 0.00% KinetiWall contention
    User->>Game: Alt+Tab or Exit Game
    Game->>WinHook: EVENT_SYSTEM_FOREGROUND (Desktop / Taskbar)
    WinHook->>Core: Callback(Desktop Expose)
    Core->>Player: IPC Command: Resume()
    Player->>Player: Resume D3D11 Present (Resume latency <=16.6ms)
```

