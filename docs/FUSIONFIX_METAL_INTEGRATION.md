# FusionFix → Theft4 native Metal integration audit (phase 1)

Status: source interface inspection; no renderer behavior changed.

## Source baselines
- FusionFix: ThirteenAG/GTAIV.EFLC.FusionFix, `source/resources/SunShafts_PS.hlsl`.
- Theft4: RoundedSpine/Theft4, `ios/bridge/theft4_metal_host_shaders.h`, `ios/bridge/theft4_metal_plan.h`.
- Keep this feature on `feature/fusionfix-metal-integration`; do not merge until visual and performance validation.

## Verified FusionFix sun-shaft inputs
- Pixel shader samples `HDRSampler` (s2), depth `GBufferTextureSampler3` (s1), cloud-mask `DiffuseTex` (s13), and downsampled HDR `pHDRDownsampleTex` (s11).
- Matrices: `gWorldViewProj` (c8), `gViewInverse` (c12); full-resolution screen size c44.
- Custom constants: `FusionPS208`, `FusionPS209`, `FusionPS210`, `FusionPS216` (c208/209/210/216).
- `SSPrepass` applies sky, sun, edge, cloud and horizon masks; `SSDraw` implements screen-space radial light scattering. Source defines 24 samples.
- The shader uses D3D9 half-pixel correction and game-specific log-depth decoding; these must not be copied blindly into Metal.

## Verified Theft4 interfaces
- `HostShaderStore` owns an offline-loaded host shader catalog, resolves shaders by name and binds declared host inputs/constants to draws.
- `HostBinding` carries binding/texture/sampler indices; `HostShaderMetadata` carries stage, specialization and constant byte count.
- `PlanAdapter::BindProduced` binds GPU-produced textures from the ordered frame; comments explicitly forbid treating them as static CPU uploads or retaining them in immutable caches.
- `PlanAdapter` has renderer-owned upload/resource caches. Avoid creating textures, shaders, or pipelines in the hot draw loop.

## Required next code inspection
1. Locate the actual host-effects frame graph, host shader metadata catalog and Metal shader source/build pipeline.
2. Verify HDR/depth/cloud textures exist simultaneously at the intended pass, and identify their Metal formats, MSAA resolve state, coordinate conventions and lifetime.
3. Identify a stable sun vector and view/projection matrices; validate whether game depth is logarithmic and how sky pixels are represented.
4. Inspect configuration/settings binding and persistence before adding any toggle.
5. Establish whether existing post-processing already provides sun shafts to prevent double application.
6. Review FusionFix source licensing and any shader-origin attribution before copying/adapting code.

## First implementation acceptance criteria
- Default OFF, no effect on vanilla rendering, no new per-frame shader compilation.
- Correct render-target ownership and synchronization; graceful disable if inputs unavailable.
- Half-resolution prepass + bounded radial gather only if compatible with render graph.
- No sun shafts through buildings; stable during camera movement, day/night, weather and resolution changes.
- Test against unchanged branch: frame-time, GPU memory, thermal behavior, visual screenshots, no crashes.
- Do not label feature 'ported' until an iOS build succeeds and on-device screenshots confirm output.

## Design decision
Prefer an optional native host effect over a Windows ASI loader. A direct HLSL copy is not yet safe because shader constants, texture bindings and depth encoding are game/renderer-specific.
