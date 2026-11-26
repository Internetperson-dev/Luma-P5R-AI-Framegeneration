# Persona 5 Royal — OptiScaler frame generation

The P5R mod now publishes its existing HUDless scene colour to OptiScaler through the NGX
parameter keys `Opti.Hudless` / `Opti.Hudless.Flip`. No HUD or UI shader path is changed:
the resource is `merged_texture`, which contains the resolved scene after the alpha merge and
before UI drawing. OptiScaler copies it at Present, after post-processing has filled it.

`CHECK_GRAPHICS_API_COMPATIBILITY` is enabled because OptiScaler's frame generation wraps the
DX11 swapchain in a DX11-on-DX12 swapchain. The core then ignores foreign D3D12 devices/command
lists instead of asserting.

## OptiScaler setup

Install Luma first, then OptiScaler as `dxgi.dll` with ReShade loaded from OptiScaler. In
`OptiScaler.ini`:

```ini
[Plugins]
LoadReShade=true

[Upscalers]
Dx11Upscaler=dlss

[FrameGen]
FGInput=Upscaler
FGOutput=FSRFG
```

Then select DLSS in Luma's Super Resolution menu as usual. `FGInput=Upscaler` is OptiScaler's
"OptiFG (Upscaler)" mode. The game-provided HUDless resource replaces the need for OptiScaler's
automatic HUDFix for this frame.

Notes:

- `FGOutput=XeFG` can also be tried if FSR FG has driver-specific issues; it is the heavier but
  generally better HUD option in OptiScaler.
- P5R's known 120 FPS cap still applies unless an external frame-rate unlocker is used.
- Without OptiScaler, the new parameter writes are ignored by NGX and behaviour is unchanged.
