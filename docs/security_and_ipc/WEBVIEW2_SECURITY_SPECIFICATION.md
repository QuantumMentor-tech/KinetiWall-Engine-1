# WEBVIEW2 NATIVE-BRIDGE SECURITY SPECIFICATION
## Document ID: KW-SEC-006 | Phase 2: System Architecture & Technical Decisions

---

### 1. Web Wallpaper Sandboxing Overview

Interactive live wallpapers execute untrusted HTML5, JavaScript, Canvas, and WebGL content. To guarantee that a downloaded wallpaper cannot access personal files, run malicious executables, or snoop on private audio, KinetiWall executes all web content inside a hardened Microsoft Edge **WebView2** runtime.

The Web Player host process (`kinetiwall-player-web.exe`) enforces **Hermetic Isolation**:

```
+--------------------------------------------------------------------------------------------------------+
|                                    WEBVIEW2 HERMETIC ISOLATION MODEL                                  |
+---------------------------------------------------+----------------------------------------------------+
|               ALLOWED CAPABILITIES                |           COMPLETELY PROHIBITED ACTIONS            |
+---------------------------------------------------+----------------------------------------------------+
| - Render 2D Canvas, WebGL 1.0/2.0, CSS Animations | - Read or write host filesystem (No file:/// access)|
| - Receive JSON wallpaper properties via bridge    | - Spawn native processes (No cmd, PowerShell, Win32)|
| - Receive downsampled FFT audio frequency buckets | - Access microphone or capture raw audio PCM       |
| - Emit rendering readiness and performance metrics| - Open external browser windows or new tabs        |
| - Local asset loading via https://kinetiwall.local| - Navigate away to external websites or IP addresses|
+---------------------------------------------------+----------------------------------------------------+
```

---

### 2. Virtual Host Mapping & Filesystem Containment

Direct access to the local filesystem via `file:///` URLs exposes severe information disclosure and cross-origin security risks.

#### 2.1 Virtual Host Name Mapping
KinetiWall utilizes WebView2's native `SetVirtualHostNameToFolderMapping` API to map the wallpaper's local asset directory to an isolated HTTPS origin:

```rust
// In kinetiwall-player-web.exe initialization:
let asset_path = format!("{}\\{}\\assets", base_wallpapers_dir, wallpaper_id);

core_webview2.SetVirtualHostNameToFolderMapping(
    w!("kinetiwall.local"),
    &HSTRING::from(asset_path),
    COREWEBVIEW2_HOST_RESOURCE_ACCESS_KIND_ALLOW,
)?;

// Navigate to virtual origin:
core_webview2.Navigate(w!("https://kinetiwall.local/index.html"))?;
```

#### 2.2 Security Enforcements of Virtual Host Mapping
1. **Zero Path Traversal**: Attempts to access `https://kinetiwall.local/../../sensitive.txt` are blocked by WebView2's internal HTTP URL parser.
2. **CORS Isolation**: The wallpaper executes under the `https://kinetiwall.local` origin. It cannot read cookies, local storage, or data from other web domains or other wallpapers.
3. **`file://` Protocol Disabled**: Any attempt to load `file:///C:/...` is blocked by the runtime.

---

### 3. Strict Navigation & Permission Lockdown

To prevent web wallpapers from transforming into rogue web browsers or phishing vectors, all navigation and permission events are locked down:

#### 3.1 Event Handler Lockdown Specifications

```rust
// 1. Block all external navigations
webview.NavigationStarting(|_, args| {
    let mut uri = PWSTR::null();
    args.Uri(&mut uri)?;
    let uri_str = uri.to_string()?;
    
    // Only permit local virtual origin
    if !uri_str.starts_with("https://kinetiwall.local/") {
        args.SetCancel(true)?; // Block navigation
        log::warn!("Blocked unauthorized navigation attempt to: {}", uri_str);
    }
    Ok(())
})?;

// 2. Block all new window requests (window.open, target="_blank")
webview.NewWindowRequested(|_, args| {
    args.SetHandled(true)?; // Discard new window request completely
    Ok(())
})?;

// 3. Deny all device permissions unconditionally
webview.PermissionRequested(|_, args| {
    args.SetState(COREWEBVIEW2_PERMISSION_STATE_DENY)?;
    Ok(())
})?;
```

#### 3.2 Denied Permissions Catalog
The following WebView2 permissions are **unconditionally set to `DENY`**:
- `COREWEBVIEW2_PERMISSION_KIND_CAMERA`
- `COREWEBVIEW2_PERMISSION_KIND_MICROPHONE`
- `COREWEBVIEW2_PERMISSION_KIND_GEOLOCATION`
- `COREWEBVIEW2_PERMISSION_KIND_NOTIFICATIONS`
- `COREWEBVIEW2_PERMISSION_KIND_OTHER_SENSORS`
- `COREWEBVIEW2_PERMISSION_KIND_CLIPBOARD_READ`

---

### 4. Content Security Policy (CSP) Injection

Every HTML document served through the virtual host mapping has a strict Content Security Policy injected via HTTP response headers:

```http
Content-Security-Policy: default-src 'self' https://kinetiwall.local; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline'; img-src 'self' data: blob:; media-src 'self' data: blob:; connect-src 'self'; object-src 'none'; frame-src 'none'; base-uri 'none'; form-action 'none';
```

- `object-src 'none'`: Prohibits Flash, Java applets, and legacy ActiveX plugins.
- `frame-src 'none'`: Prevents embedding hidden external phishing frames.
- `connect-src 'self'`: Disallows outbound network telemetry and data exfiltration to external domains.

---

### 5. Native Bridge Architecture & Audio Spectrum Delivery

#### 5.1 Host Object Prohibition
KinetiWall **strictly bans `AddHostObjectToScript`**. No native C++ or Rust structs are injected into the JavaScript execution context. This completely removes the attack surface of COM method hijacking and memory corruption from JavaScript.

#### 5.2 The Constrained postMessage API
All communication between the Web Player host and the wallpaper script occurs through string-based JSON serialization over `window.chrome.webview.postMessage`:

```javascript
// Web Wallpaper JavaScript Interface
window.addEventListener("message", (event) => {
    const msg = event.data;
    switch (msg.type) {
        case "PROPERTIES_UPDATED":
            applyCustomProperties(msg.properties);
            break;
        case "AUDIO_SPECTRUM":
            renderAudioVisualizer(msg.frequencies);
            break;
    }
});

// Wallpaper reporting readiness
window.chrome.webview.postMessage(JSON.stringify({
    type: "WALLPAPER_READY",
    width: window.innerWidth,
    height: window.innerHeight
}));
```

#### 5.3 Audio Visualization Privacy Pipeline
To satisfy **KW-F-024** and prevent user surveillance:
1. **Zero Raw PCM Exposure**: Web wallpapers NEVER receive raw audio audio buffers.
2. **Frequency Bucketing**: The Core Controller's DSP pipeline converts audio into 16 or 32 normalized floating-point numbers ($0.0 \le f \le 1.0$) representing discrete octave frequency bands (e.g. Bass 60Hz, Mid 1kHz, Treble 8kHz).
3. **Reversibility Impossibility**: It is mathematically impossible to reconstruct spoken words, private speech, or music audio from 16 downsampled energy buckets delivered at 30 Hz.
4. **Automatic Pause**: When system audio ceases or when the display is sleeping, spectrum streaming stops immediately, reducing CPU usage to 0%.

