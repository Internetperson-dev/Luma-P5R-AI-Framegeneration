# Persona 5 Royal — OptiScaler frame generation

The P5R mod now publishes its existing HUDless scene colour to OptiScaler through the NGX
parameter keys `Opti.Hudless` / `Opti.Hudless.Flip`. No HUD or UI shader path is changed:
the resource is `merged_texture`, which contains the resolved scene after the alpha merge and
before UI drawing. OptiScaler copies it at Present, after post-processing has filled it.

`CHECK_GRAPHICS_API_COMPATIBILITY` is enabled because OptiScaler's frame generation wraps the
DX11 swapchain in a DX11-on-DX12 swapchain. The core then ignores foreign D3D12 devices/command
lists instead of asserting.

## OptiScaler setup

Install Luma first, then OptiScaler as `dxgi.dll` with ReShade loaded from OptiScaler. Open Luma's
**Settings** tab and use the new **OptiScaler integration** section:

| Luma option | What it controls |
|---|---|
| Mode | `Off`, `Upscaling only`, `Frame generation only`, `Upscaling + frame generation` |
| FG backend | FSR FG, DLSS FG or XeSS FG (`[FrameGen] FGOutput`) |
| OptiScaler upscaler | The SR model OptiScaler substitutes for Luma's NGX call (`[Upscalers] Dx11Upscaler`) |
| Low latency | `Off`, `Auto`, NVIDIA Reflex, AMD Anti-Lag 2 or Intel XeLL (`[fakenvapi]`) |

Luma writes these choices to `OptiScaler.ini` automatically and stores them in `ReShade.ini`.
Changing any of them prompts you to restart: **Yes** closes the game immediately, while **No**
keeps the saved settings for the next launch. They are applied on the next game launch. Luma
always uses `FGInput=Upscaler`, so the HUDless scene colour supplied through `Opti.Hudless` is used
instead of OptiScaler's automatic HUDFix.

The **Mode** control is the only place you need to change behaviour. Luma automatically selects
the DLSS SR dispatch that OptiScaler intercepts:

- `Off` disables both Luma SR and OptiScaler FG.
- `Upscaling only` enables SR and disables FG.
- `Frame generation only` enables both the SR dispatch and FG; with P5R rendering at output
  resolution this becomes DLAA + FG.
- `Upscaling + frame generation` enables both with normal P5R render scaling.

The **OptiScaler upscaler** option still replaces the visual SR model with FSR or XeSS while Luma
performs the NGX dispatch.

The equivalent manual configuration is:

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

Notes:

- `Auto` latency selects Reflex on NVIDIA, Anti-Lag 2 on AMD RDNA and XeLL on Intel. Anti-Lag 2
  requires an RDNA GPU; XeLL needs Intel's low-latency runtime. NVIDIA "Boost" behaviour is
  driver-side and follows an enabled Reflex mode.
- P5R's default 120 FPS cap still limits the base frame rate without an external frame-rate
  unlocker. With your uncapper installed, frame generation can add frames above that cap, provided
  the FG backend is stable on your GPU/driver.
- Without OptiScaler, the NGX parameter writes are ignored and behaviour is unchanged.
