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
