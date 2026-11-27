# Luma — Persona 5 Royal frame generation

Standalone, P5R-only build containing **Luma + OptiScaler**. It includes Luma's shared Core and
P5R addon project, with all other game projects removed, plus vendored OptiScaler source under
`Source/External/OptiScaler`. GitHub Actions builds both and packages them into a single ZIP.

Luma is the GUI/control surface. OptiScaler is the backend that performs frame generation, SR
model replacement and Reflex/Anti-Lag 2/XeLL latency handling. The addon supplies P5R's existing
SR resources (scene colour, depth, decoded motion vectors) through NGX parameters, including the
HUDless `merged_texture`; HUD/UI shader paths are untouched.

OptiScaler is GPL-3.0. Its upstream source, license and provenance are included in
`Source/External/OptiScaler`.

## Known issues

Some HDR behaviuor is broken on Proton/Linux for now. 

This causes crashes without `PROTON_DISABLE_NVAPI=1` and FSR 3 to freeze on a 3D scene.

```
XeSS status
- Your currently installed game is upstream latest-840, which has only Luma’s native DLSS/FSR SR path. That is why you only see FSR 3.
- This repo is the P5R frame-generation fork. It already exposes:
- OptiScaler upscaler = DLSS / FSR / XeSS
- FG backend = FSR FG / DLSS FG / XeSS FG
- These are implemented in Source/Games/Persona 5 Royal/main.cpp:259 and :267.
- The fork’s release p5r-11 zip already ships:
- OptiScaler/libxess.dll
- OptiScaler/libxess_dx11.dll
- OptiScaler/libxess_fg.dll
- OptiScaler/libxell.dll
So there are two different “XeSS paths”:
1. OptiScaler XeSS — already implemented and packaged in this fork.
2. Luma native XeSS — not enabled for P5R. Persona 5 Royal.vcxproj sets UseLumaNGX=true and UseLumaFSR=true, but not UseLumaXeSS. Also this standalone fork does not include the top-level Source/External/XeSS headers/bin that Luma.Features.props:371-374 expects, so it is not a one-line toggle. On Proton it would also use DX11→DX12 interop for non-Intel GPUs, which is the same risky class of path as the FSR freeze.
NVAPI / DLSS cause
The crash is not a missing runtime. It is Luma’s HDR/NVAPI display query hitting a Wine gap:
core.hpp:3002  GetHDRMaxLuminance()
  display.inl:797-799  GetHDR10PlusDisplayPeakBrightness()
    NvAPI_DISP_GetDisplayIdByDisplayName
    NvAPI_Disp_GetHdrCapabilities
core.hpp:3003  IsHDRSupportedAndEnabled()
  display.inl:503-514  GetColorInfo2()
    DisplayConfigGetDeviceInfo packet type 15
Packet type 15 is DISPLAYCONFIG_DEVICE_INFO_GET_ADVANCED_COLOR_INFO_2. Wine logs it as unimplemented, then the stack overflow follows.
PROTON_DISABLE_NVAPI=1 works because it prevents that HDR/NVAPI query chain from running at all. I checked current dxvk-nvapi source: there is no narrow “disable HDR only” environment switch. So the targeted Luma-side fix is to skip the HDR10+ peak-brightness query and the packet-15 advanced color query under Wine, while leaving NVAPI loaded for DLSS.
That means:
- DLSS path: patch Source/Core/utils/display.inl with an isWine guard around the NVAPI HDR block and the GetColorInfo2 call. Keep NVAPI enabled.
- XeSS path: install this fork’s p5r-11 release and choose XeSS in the OptiScaler integration panel. No new code needed.
I have not changed any files. If you want, I can implement the Wine-only HDR/NVAPI guard next so DLSS can run without PROTON_DISABLE_NVAPI=1, or help you switch the install over to the fork’s XeSS-enabled build.
Build · Big Pickle · 6m 22s · 212.9 tok/s
↳ Shell finished · python - <<'PY' import json, urllib.request, re for tag in ['v0.6.4','v0.7.0','v0.7.1','v0.8.0','v0.9.0','v0.9.1','v0.9.2','v0.9.3']: base=f'https://raw.githubusercontent.com…
+1 thought
The background check finished and changes one detail: none of the dxvk-nvapi versions in these Proton builds contain any advanced-color/DisplayConfigGetDeviceInfo call. So packet type 15 is coming from Luma’s own GetColorInfo2(), not from DXVK-NVAPI. I’m reconciling that with why the NVAPI disable flag still avoids the crash.
+5 commands, 13 thoughts, 3 reads, 7 tools
That background result is useful, and I checked the current working log too. It changes the diagnosis slightly:
- None of the dxvk-nvapi versions in your Proton builds call DisplayConfigGetDeviceInfo.
- The current working log contains NtUserDisplayConfigGetDeviceInfo Unimplemented packet type 15 many times and does not crash.
So packet type 15 is not fatal by itself. It is Luma’s GetColorInfo2() at display.inl:503-514, and Wine simply returns failure so Luma falls back to the legacy query at display.inl:563-565.
Revised NVAPI diagnosis
The fatal sequence is the NVAPI HDR10+ peak-brightness path, not the packet-15 fixme on its own:
core.hpp:3002
  GetHDRMaxLuminance()
    display.inl:797-799
      GetHDR10PlusDisplayPeakBrightness()
        NvAPI_DISP_GetDisplayIdByDisplayName
        NvAPI_Disp_GetHdrCapabilities
With PROTON_DISABLE_NVAPI=1, NvAPI_Initialize() fails, NVAPI::hasInit stays false, and that whole block is skipped. Packet 15 still happens harmlessly.
I also found no dxvk-nvapi environment switch that disables only HDR queries. So the correct narrow fix is in Luma, not Proton:
// In GetHDRMaxLuminance()
static const bool isWine =
    GetProcAddress(GetModuleHandleW(L"ntdll.dll"), "wine_get_version") != nullptr;

#if ENABLE_NVAPI
if (!isWine && NVAPI::hasInit && !HasUserHDRColorProfile(0, outputDesc1.Monitor))
{
    // HDR10+ peak brightness query
}
#endif
Optionally also guard the HDR10+ enable at core.hpp:3005-3009:
#if ENABLE_NVAPI
if (!isWine && hdr_enabled_display)
{
    Display::NVAPI::SetDisplayOutputMode(...);
}
#endif
This keeps NVAPI available for DLSS while skipping the Wine-hostile HDR calls.
DLSS / XeSS practical options
1. DLSS through this fork’s OptiScaler path  
The p5r-11 package bundles nvngx_dlss.dll and does not ship nvapi64.dll. That path should work with PROTON_DISABLE_NVAPI=1 already.
2. Luma native DLSS with real NVAPI  
Use the Wine guard above, then remove PROTON_DISABLE_NVAPI=1.
3. XeSS  
Already implemented in this fork through OptiScaler. The release package contains libxess.dll, libxess_dx11.dll, libxess_fg.dll, and libxell.dll. Native Luma XeSS for P5R is not currently buildable here because the standalone fork lacks the top-level Source/External/XeSS dependency.
No files have been changed. If you want, I can implement the Wine-only HDR/NVAPI guard next.
Build · Big Pickle · 2m 6s · 214.0 tok/s
NVAPI / DLSS cause
The crash is not a missing runtime. It is Luma’s HDR/NVAPI display query hitting a Wine gap:
core.hpp:3002  GetHDRMaxLuminance()
  display.inl:797-799  GetHDR10PlusDisplayPeakBrightness()
    NvAPI_DISP_GetDisplayIdByDisplayName
    NvAPI_Disp_GetHdrCapabilities
core.hpp:3003  IsHDRSupportedAndEnabled()
  display.inl:503-514  GetColorInfo2()
    DisplayConfigGetDeviceInfo packet type 15
Packet type 15 is DISPLAYCONFIG_DEVICE_INFO_GET_ADVANCED_COLOR_INFO_2. Wine logs it as unimplemented, then the stack overflow follows.  any way to skip over if the user is in wine with a warning saying it is broken for now?  You can implement it at /home/ram/Projects/Luma-P5R
+1 thought
I’ll implement a Wine guard in the Luma display path: skip the HDR10+/NVAPI query and the packet-15 advanced-color query under Wine, log one warning, and fall back to Wine’s supported legacy display query. I’ll keep native Windows behavior unchanged.
+1 edit, 4 thoughts, 3 reads, 4 tools
```

## Build on Windows

Requirements:

- Windows 11 (Windows 10 likely works)
- Visual Studio 2022 or 2026 with the C++ desktop workload
- Windows 11 SDK (`10.0.26100.0` recommended)

Open `Luma.sln`, select an `x64` configuration, and build. The project uses the v143 toolset with VS2022 and automatically switches to v145 under VS2026.

Equivalent command line:

```bat
msbuild Source\External\OptiScaler\OptiScaler.sln /m /p:Configuration=Release /p:Platform=x64
msbuild Luma.sln /m /p:Configuration=Development-Release /p:Platform=x64
```

Packaging (after both builds):

```bat
pwsh -ExecutionPolicy Bypass -File Scripts/package.ps1 -Project "Persona 5 Royal" -Config "Development-Release" -Platform x64 -OptiScalerDir "Source\External\OptiScaler\x64\Release\a"
```

The ZIP then contains OptiScaler as `dxgi.dll`, ReShade as `ReShade64.dll`, the Luma addon, shaders and runtime DLLs. GitHub Actions builds and packages all of this on Windows for `Development-Release`, `Test-Release`, and `Publishing-Release`.

## Frame rate unlocker (recommended)

Frame generation is most useful when P5R is not limited by its default 120 FPS cap, so using a
frame rate unlocker is recommended. This fork is intended to be used with
[p5rpc.fpsUnlocker](https://github.com/Internetperson-dev/p5rpc.fpsUnlocker). Without an unlocker,
the game's default cap still limits the base frame rate before frame generation is applied.

## OptiScaler setup

See [`Source/Games/Persona 5 Royal/OPTISCALER_FG.md`](Source/Games/Persona%205%20Royal/OPTISCALER_FG.md).

Extract the release ZIP into the P5R game folder. Do **not** install a separate OptiScaler build:
the package already contains OptiScaler as `dxgi.dll` and ReShade as `ReShade64.dll`.

Luma's **Settings** tab is now the only control surface. Its **OptiScaler integration** section
exposes:

- Mode: Off / Upscaling only / Frame generation only / Upscaling + frame generation
- FG backend: FSR FG / DLSS FG / XeSS FG
- OptiScaler upscaler: DLSS / FSR / XeSS
- Low latency: Off / Auto / Reflex / Anti-Lag 2 / XeLL

Luma automatically selects the DLSS SR dispatch that OptiScaler intercepts, so the separate Luma
Super Resolution menu does not need to be touched. Choices are saved in `ReShade.ini` and written
to `OptiScaler.ini` automatically. Luma prompts to restart the game when any of these options
change; choosing **Yes** closes the game immediately, while **No** keeps the saved settings for the
next launch. The equivalent manual `OptiScaler.ini` settings are:

```ini
[Plugins]
LoadReShade=true

[Inputs]
EnableDlssInputs=true

[Upscalers]
Dx11Upscaler=dlss

[Hotfix]
CreateD3D12DeviceForLuma=true

[Dx11withDx12]
DontUseNTShared=true

[FrameGen]
Enabled=true
FGInput=Upscaler
FGOutput=FSRFG
DisableHudless=false

[fakenvapi]
UseFakenvapi=true
LowLatencyInput=1
LowLatencyOutput=1
ForceReflex=0
```

## Known Linux/Proton limitation: Luma HDR10+ queries

Luma's HDR10+ display queries are currently skipped under Wine/Proton. Wine does not implement
`DISPLAYCONFIG_DEVICE_INFO_GET_ADVANCED_COLOR_INFO_2` (packet type 15), and Luma's HDR10+
peak-brightness / output-mode calls through NVAPI can crash with dxvk-nvapi.

This does **not** mean all HDR on Linux is broken. Proton/DXVK can still present HDR through DXGI,
and games that only use swapchain HDR color spaces or the legacy display query may work. The problem
is specific to Luma's Windows advanced-color and NVAPI HDR10+ display-control path, especially with
P5R's swapchain behavior.

Luma now detects Wine/Proton at runtime, logs a one-time warning, and skips those HDR/NVAPI display
queries. It falls back to the legacy display-state query where available. This does **not** disable
NVAPI globally, so DLSS remains available; `PROTON_DISABLE_NVAPI=1` is no longer required for this
specific crash. SDR gameplay, upscaling and frame generation are unaffected. Luma's native HDR10+
toggling and peak-brightness detection remain unsupported on Linux until Wine/Proton implement the
required display APIs.
