# Luma — Persona 5 Royal frame generation

Standalone, P5R-only fork of Luma. It contains the shared Core and the P5R addon project, with all other game projects removed, so builds only produce:

`Luma-Persona 5 Royal.addon`

The addon exposes P5R's existing Luma SR resources (scene colour, depth, decoded motion vectors) to OptiScaler's frame-generation path through NGX parameters. The HUD/UI shader paths are untouched.

## Build on Windows

Requirements:

- Windows 11 (Windows 10 likely works)
- Visual Studio 2022 or 2026 with the C++ desktop workload
- Windows 11 SDK (`10.0.26100.0` recommended)

Open `Luma.sln`, select an `x64` configuration, and build. The project uses the v143 toolset with VS2022 and automatically switches to v145 under VS2026.

Equivalent command line:

```bat
msbuild Luma.sln /m /p:Configuration=Development-Release /p:Platform=x64
```

Packaging (after a build):

```bat
pwsh -ExecutionPolicy Bypass -File Scripts/package.ps1 -Project "Persona 5 Royal" -Config "Development-Release" -Platform x64
```

GitHub Actions builds and packages P5R on Windows for `Development-Release`, `Test-Release`, and `Publishing-Release`.

## OptiScaler setup

See [`Source/Games/Persona 5 Royal/OPTISCALER_FG.md`](Source/Games/Persona%205%20Royal/OPTISCALER_FG.md).

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
