# THIRD-PARTY LICENSE COMPLIANCE & REDISTRIBUTION REGISTER
## Document ID: KW-DEP-003 | Phase 2: System Architecture & Technical Decisions

---

### 1. License Categorization & Compliance Tiers

To prevent unintended copyright infringement, legal disputes, or viral copyleft license contamination, all third-party software components are classified into four strict compliance tiers:

```
+--------------------------------------------------------------------------------------------------------------------------+
|                                           KINETIWALL LICENSE COMPLIANCE TIERS                                            |
+--------+--------------------------+-----------------------+--------------------------------------------------------------+
| TIER   | CLASSIFICATION           | REPRESENTATIVE SPDX   | RELEASING OBLIGATIONS & ARCHITECTURAL BOUNDARIES             |
+--------+--------------------------+-----------------------+--------------------------------------------------------------+
| Tier 1 | Permissive (Green)       | MIT, Apache-2.0, ISC, | - Distribute copyright notices and license text in notices.  |
|        |                          | BSD-2/3-Clause        | - Zero source code disclosure obligations.                   |
| Tier 2 | Weak Copyleft (Yellow)   | LGPL-2.1-or-later,    | - Permitted ONLY as dynamically linked DLL or separate proc. |
|        |                          | MPL-2.0               | - Modifications to library source MUST be disclosed.         |
|        |                          |                       | - Must permit user to replace/relink the library.            |
| Tier 3 | Strong Copyleft (Red)    | GPL-2.0, GPL-3.0,     | - PROHIBITED from static linking into KinetiWall core.       |
|        |                          | AGPL-3.0              | - Permitted ONLY across separate process IPC boundaries.     |
| Tier 4 | Proprietary / Commercial | Microsoft EULA        | - Permitted only under explicit royalty-free distribution    |
|        |                          | (WebView2, VC++ CRT)  |   redistribution agreements (e.g. Microsoft Visual Studio).  |
+--------+--------------------------+-----------------------+--------------------------------------------------------------+
```

---

### 2. Comprehensive Third-Party License Register

| Dependency ID | Name | Ecosystem | Declared SPDX License | Compliance Tier | Primary Distribution Obligation | Verified License Evidence |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **DEP-01** | `windows` | Cargo | `MIT OR Apache-2.0` | Tier 1 (Green) | Include notice in `THIRD_PARTY_NOTICES.txt` | [GitHub License](https://github.com/microsoft/windows-rs/blob/master/LICENSE-MIT) |
| **DEP-02** | `windows-sys` | Cargo | `MIT OR Apache-2.0` | Tier 1 (Green) | Include notice | [GitHub License](https://github.com/microsoft/windows-rs/blob/master/LICENSE-MIT) |
| **DEP-03** | `tokio` | Cargo | `MIT` | Tier 1 (Green) | Include copyright notice | [Tokio License](https://github.com/tokio-rs/tokio/blob/master/LICENSE) |
| **DEP-04** | `tray-icon` | Cargo | `MIT OR Apache-2.0` | Tier 1 (Green) | Include notice | [GitHub License](https://github.com/tauri-apps/tray-icon/blob/dev/LICENSE-MIT) |
| **DEP-06** | `serde` / `serde_json` | Cargo | `MIT OR Apache-2.0` | Tier 1 (Green) | Include notice | [Serde License](https://github.com/serde-rs/serde/blob/master/LICENSE-MIT) |
| **DEP-07** | `uuid` | Cargo | `MIT OR Apache-2.0` | Tier 1 (Green) | Include notice | [GitHub License](https://github.com/uuid-rs/uuid/blob/main/LICENSE-MIT) |
| **DEP-08** | `libmpv` (`mpv-2.dll`)| Native | `LGPL-2.1-or-later` | **Tier 2 (Yellow)** | **Dynamic link ONLY; provide source offer** | [mpv Copyright](https://github.com/mpv-player/mpv/blob/master/Copyright) |
| **DEP-10** | `WebView2 Runtime` | Windows| Proprietary EULA | **Tier 4 (Blue)** | Distribute via Microsoft Evergreen installer | [WebView2 Terms](https://developer.microsoft.com/en-us/microsoft-edge/webview2/) |
| **DEP-11** | `webview2-com` | Cargo | `MIT OR Apache-2.0` | Tier 1 (Green) | Include notice | [GitHub License](https://github.com/wravery/webview2-rs/blob/master/LICENSE-MIT) |
| **DEP-12** | `tauri` v2 | Cargo | `MIT OR Apache-2.0` | Tier 1 (Green) | Include notice | [Tauri License](https://github.com/tauri-apps/tauri/blob/dev/LICENSE_MIT) |
| **DEP-13** | `@tauri-apps/api`| npm | `MIT OR Apache-2.0` | Tier 1 (Green) | Include notice | [npm License](https://www.npmjs.com/package/@tauri-apps/api) |
| **DEP-14** | `react` / `react-dom`| npm | `MIT` | Tier 1 (Green) | Include Meta copyright notice | [React License](https://github.com/facebook/react/blob/main/LICENSE) |
| **DEP-16** | `tailwindcss` | npm | `MIT` | Tier 1 (Green) | Include notice in CSS bundles | [Tailwind License](https://github.com/tailwindlabs/tailwindcss/blob/master/LICENSE) |
| **DEP-17** | `lucide-react` | npm | `ISC` | Tier 1 (Green) | Include ISC copyright notice | [Lucide License](https://github.com/lucide-icons/lucide/blob/main/LICENSE) |
| **DEP-22** | `cpal` | Cargo | `Apache-2.0` | Tier 1 (Green) | Include Apache notice + State changes | [CPAL License](https://github.com/rust-audio/cpal/blob/master/LICENSE) |
| **DEP-23** | `rustfft` | Cargo | `MIT OR Apache-2.0` | Tier 1 (Green) | Include notice | [RustFFT License](https://github.com/ejmahler/RustFFT/blob/master/LICENSE-MIT) |
| **DEP-24** | `image` | Cargo | `MIT OR Apache-2.0` | Tier 1 (Green) | Include notice | [Image License](https://github.com/image-rs/image/blob/master/LICENSE-MIT) |
| **DEP-25** | `zip` | Cargo | `MIT` | Tier 1 (Green) | Include notice | [Zip License](https://github.com/zip-rs/zip2/blob/master/LICENSE) |
| **DEP-26** | `sha2` | Cargo | `MIT OR Apache-2.0` | Tier 1 (Green) | Include notice | [RustCrypto License](https://github.com/RustCrypto/hashes/blob/master/sha2/LICENSE-MIT) |
| **DEP-27** | `tracing` | Cargo | `MIT` | Tier 1 (Green) | Include notice | [Tracing License](https://github.com/tokio-rs/tracing/blob/master/LICENSE) |

---

### 3. The Critical libmpv & FFmpeg Licensing Boundary

Video playback via `mpv` represents the single most significant legal compliance requirement in KinetiWall. Upstream mpv can be compiled under either **LGPL-2.1-or-later** or **GPL-2.0-or-later** depending on build-time configuration flags:

#### 3.1 Build Configuration & Flag Mandate
To ensure KinetiWall can be distributed cleanly under standard commercial and permissive terms, any prebuilt `mpv-2.dll` bundled with KinetiWall MUST be built strictly under **LGPL-2.1-or-later**:

```bash
# MANDATORY MPV BUILD FLAGS FOR KINETIWALL DISTRIBUTIONS:
meson setup build \
    --default-library=shared \
    --enable-lgpl \
    -Dgpl=false \
    -Dcplayer=false \
    -Dlibmpv=true
```

**Consequences of the `--enable-lgpl` / `-Dgpl=false` Flags**:
1. All GPL-licensed modules (such as GPL filters and optional GPL post-processing algorithms) are excluded from the binary.
2. The underlying FFmpeg build MUST also be compiled with `--enable-version3 --disable-gpl`.
3. The resulting `mpv-2.dll` is legally licensed under **LGPL-2.1-or-later**.

#### 3.2 Dual Defense-in-Depth Isolation (Process Boundary Immunity)
Even if a user manually replaces `mpv-2.dll` with a custom GPL-compiled build (e.g. to enable custom GPL video filters):
- `mpv-2.dll` is dynamically loaded **strictly within the isolated subprocess `kinetiwall-player-video.exe`**.
- It is NEVER loaded or linked into `kinetiwall-core.exe` or `kinetiwall-ui.exe`.
- Communication between the Core and Player takes place strictly across standard operating system inter-process communication boundaries (Windows Named Pipes).
- Under Free Software Foundation (FSF) licensing jurisprudence, separate programs communicating over IPC at arm's length do not form a single combined derivative work, protecting `kinetiwall-core.exe` from reciprocal copyleft contamination.

#### 3.3 LGPL Compliance Requirements Checklist
For the bundled `mpv-2.dll`:
- [x] **Dynamic Linking Only**: KinetiWall dynamically links `mpv-2.dll`. No static linking.
- [x] **User Relinking Rights**: Users are legally and technically free to replace `bin\mpv-2.dll` with their own compatible version; KinetiWall will dynamically load it on next launch.
- [x] **Source Offer**: KinetiWall provides a written offer (in `THIRD_PARTY_NOTICES.txt`) and a GitHub release link to the exact source code, build scripts, and Meson configuration used to compile the distributed `mpv-2.dll`.

---

### 4. Software Patent & Codec Considerations

Software patents covering video codecs (notably AVC/H.264 and HEVC/H.265) pose licensing questions for media player distributors:
1. **Hardware Acceleration Offload (D3D11VA / DXVA2)**:
   KinetiWall configures `mpv` to default to Direct3D11 Video Acceleration (`--hwdec=d3d11va`). Hardware decoding utilizes the licensed hardware decoder silicone built into the user's Intel, AMD, or NVIDIA GPU (and licensed via the GPU vendor or Windows OS), eliminating software decoding patent royalty liabilities.
2. **Royalty-Free Codec Preference**:
   KinetiWall natively supports and encourages royalty-free next-generation open media formats (AV1, VP9, Opus) which carry zero patent pool licensing restrictions.

---

### 5. Binary and Runtime Distribution Register (Deliverable E)

| Binary Artifact | Version / Build | Target Architecture | License | Distributable Package Location | Provenance / Official Source | Integrity Checksum (SHA-256) |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| `mpv-2.dll` | 0.38.0 (LGPL) | `x86_64-pc-windows-msvc` | LGPL-2.1+ | `bin/mpv-2.dll` | Official KinetiWall LGPL CI Build | `TBD_AT_RELEASE_BUILD` |
| `mpv-2.dll` | 0.38.0 (LGPL) | `aarch64-pc-windows-msvc` | LGPL-2.1+ | `bin/arm64/mpv-2.dll` | Official KinetiWall LGPL CI Build | `TBD_AT_RELEASE_BUILD` |
| `WebView2Loader.dll` | 1.0.2592.51 | `x86_64-pc-windows-msvc` | MS EULA | `bin/WebView2Loader.dll` | Microsoft Official NuGet / SDK | `TBD_AT_RELEASE_BUILD` |
| `vcruntime140.dll` | 14.40.33810 | `x86_64-pc-windows-msvc` | MS EULA | Bundled / VCRedist | Microsoft Visual C++ 2015-2022 | OS / Official Microsoft Installer |
| `msvcp140.dll` | 14.40.33810 | `x86_64-pc-windows-msvc` | MS EULA | Bundled / VCRedist | Microsoft Visual C++ 2015-2022 | OS / Official Microsoft Installer |

---

### 6. Product Branding & Trademark Governance

1. **"KinetiWall" Trademark**:
   The name "KinetiWall" and associated branding assets represent proprietary project marks. While engine code is released under permissive open-source terms, third parties redistributing modified forks must rename the application to prevent user confusion.
2. **Logo Assets (`KinetiWall Neon Glass Logo.png`)**:
   The application logo located at the repository root is owned exclusively by the KinetiWall project. Re-use in commercial forks requires written authorization.
3. **Third-Party Icons**:
   UI icons sourced from `lucide-react` are governed by the **ISC license**, permitting full commercial and personal redistribution provided the ISC copyright notice is preserved in attribution files.

