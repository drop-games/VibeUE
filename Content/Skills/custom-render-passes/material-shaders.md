---
name: custom-render-passes/material-shaders
description: Converting a Global Shader to a Material Shader (FMaterialShader) so a Material graph drives its parameters — SetShaderParametersMixed*, simulation vs. screen-space pass types, HLSL Material.ush inclusion, the PostConfigInit load-phase requirement, and known limitations (async compile, DX12 warnings, viewport dependency).
---

# Material Shaders

Materials aren't shaders — they're "a common language for the tech-artist to provide code and data to
the graphics programmer." A Material describes physical properties (Base Color, Roughness, Metallic,
World Position Offset, or — for custom passes — whatever custom pins you expose, see
`custom-material-io.md`) without knowing how the pass consumes them. This is what lets a framework
built once (this whole skill's subject) absorb new artist-driven effects without new C++.

Contrast with Unity URP, where a custom shader must itself provide a pass for every render feature it
wants to support: Unreal's Material system decouples iterative tech-art work from dense engineering
work, and a new engine feature doesn't force every existing Material to be revisited.

## Migrating FGlobalShader → FMaterialShader

- Change the base class from `FGlobalShader` to `FMaterialShader`.
- Change `SHADER_USE_PARAMETER_STRUCT` to `SHADER_USE_PARAMETER_STRUCT_WITH_LEGACY_BASE` — this
  suppresses assert failures when a Material instance's parameters don't exactly match your struct's
  fields (which they won't, since the Material graph adds its own).
- Replace manual `SetShaderParameters()` calls with `SetShaderParametersMixed[VS|PS|CS|...]()` — this
  uploads **both** your parameter struct **and** the shader's inherited Material `SetParameters()`
  member. Skipping this leaves Material-sourced parameters at garbage values.
- Extract shaders from the **Material's own shader-map**, not the global shader-map. The series'
  helper library exposes this as `EGP::FindMaterialShaders_RenderThread()`, which also implements a
  fallback chain through default Materials when the target Material's shader hasn't compiled yet.

## 🚨 Load-phase requirement

Material Shader modules must load at **`PostConfigInit`**. Unlike Global Shaders (which fail loudly
at startup if their shader-map is missing), a missing Material shader-map in a **standalone build**
produces sporadic crashes later — not a startup error pointing at the cause. Get the `.uplugin`
`LoadingPhase` right up front; see `setup-and-debugging.md`.

## Pass types (Post-Process Material domain)

**Simulation passes** — run off-screen, no viewport data conceptually needed:
- Inherit `EGP::FSimulationShader`.
- Parameter-struct macro: `EGP_SIMULATION_PASS_MATERIAL_DATA()`.
- Input struct: `EGP::FSimulationPassMaterialInputs`; state struct: `EGP::FSimulationPassState`.
- Dispatch: `EGP::AddSimulationMaterialPass()`.
- Current limitation: must still pass an `FViewInfo` despite being conceptually viewport-independent
  — reduces the design win over a Screen-Space pass; may improve in future engine versions.

**Screen-space passes** — two shader-frequency shapes:
- Vertex+Pixel: inherit `EGP::FScreenSpaceShader`, use `EGP::FScreenSpacePassRenderState`, dispatch
  `EGP::AddScreenSpaceRenderPass()`. The built-in `EGP::FScreenSpaceRenderVS` covers the vertex stage
  in the common case.
- Compute: use `EGP::FScreenSpacePassComputeState`, dispatch `EGP::AddScreenSpaceComputePass()`.
- Input struct: `EGP::FScreenSpacePassMaterialInputs`.

(`EGP::*` names are the series' own helper library — treat them as *the shape of the pattern to
build*, not literal engine API; there is no first-party Unreal equivalent shipped out of the box.)

## HLSL structure

```hlsl
#include "/EGP/ScreenPass/pre.ush"
#define GOL_PASS_INIT 1   // pass-identification macro(s), pass-specific
#include "/EGP/ScreenPass/post.ush"
```
This sequence is what injects the generated Material code. For IDE intellisense without breaking the
real include:
```hlsl
#if defined(__INTELLISENSE__) || defined(__RESHARPER__)
  #include "/Engine/Private/MaterialTemplate.ush"
#else
  #include "/Engine/Generated/Material.ush"
#endif
```

### Per-frequency setup calls
- **Simulation compute shader**: manually fabricate `FPixelMaterialInputs`/`FMaterialPixelParameters`
  from thread indices (compute UV/world-position/depth yourself), then
  `ScreenPassSetupCS(viewportPixel, uv, worldPos, depth, mInputs, mParams)`.
- **Screen-space compute shader**: simpler — `ScreenPassSetupCS(pixel, mInputs, mParams)`, leaning on
  viewport data that's already available.
- **Screen-space vertex shader**: fill `FMaterialVertexParameters`, populate
  `FTexCoordsVSToPS materialInterpolants` for the pixel stage, call
  `ScreenPassSetupVS(uv, mParams, materialInterpolants)`.
- **Screen-space pixel shader**: receive `FTexCoordsVSToPS materialInputsFromVS` from the vertex
  stage, require `SV_Position`, call
  `ScreenPassSetupPS(svPos, materialInputsFromVS, matInputs, matParams)`.

### Reading Material outputs
`GetMaterialEmissive(matInputs)`, `GetMaterialBaseColor()`, etc. — generated functions for the
standard Material pins. Interpret the return semantically per pass (e.g. RGB channels as three
thresholds, a single channel as a bool via `step(0.5, value)`).

## Shader class boilerplate

```cpp
DECLARE_EXPORTED_SHADER_TYPE(FMyShaderName, Material, )
BEGIN_SHADER_PARAMETER_STRUCT(FParameters, )
    EGP_SCREEN_SPACE_PASS_MATERIAL_DATA()   // or EGP_SIMULATION_PASS_MATERIAL_DATA()
    RENDER_TARGET_BINDING_SLOTS()
    // ... custom SHADER_PARAMETER_* fields ...
END_SHADER_PARAMETER_STRUCT()
SHADER_USE_PARAMETER_STRUCT_WITH_LEGACY_BASE(FMyShader, EGP::FScreenSpaceShader)
```
```cpp
IMPLEMENT_MATERIAL_SHADER_TYPE(, FMyShaderName, TEXT("/Path/To/Shader.usf"), TEXT("EntryPoint"), SF_Pixel);
// SF_Compute for compute shaders, SF_Vertex for vertex shaders
```
Inject compile-time constants (e.g. compute group sizes) via `ModifyCompilationEnvironment()`:
`env.SetDefine(TEXT("GROUP_SIZE_X"), GroupSize().X);`

### Post-Process domain specifics
Provides multi-texture input access (`PostProcessInput_0_Texture` … `PostProcessInput_4_Texture`),
standard scene-texture access, and viewport data. A vertex+pixel render pass's pixel-shader parameter
struct nests the vertex parameters as `ScreenSpacePassData` — extract with
`&pixelParams->ScreenSpacePassData` when using the built-in vertex shader.

## Dispatch pattern (5 steps)
1. Inherit from the right `EGP::F*Shader` subclass.
2. Add the pass macro to the parameter struct.
3. Build the Material-inputs instance (`FSimulationPassMaterialInputs` / `FScreenSpacePassMaterialInputs`).
4. Build a state instance (work-group counts — set immediately, via deferred render-thread lambda, or
   via indirect GPU-buffer dispatch — plus render state).
5. Call the dispatch function with all of the above plus the target `UMaterialInterface*`.

## Known limitations

- **Async compile**: Material shaders compile on first use, not at startup — a freshly-authored effect
  runs against a Default Material (all pins unwritten) until compilation finishes. Workaround used in
  the series: re-run an "Initialize" pass a couple of seconds after game start
  (`ReInitializeAllViews()` triggered from Game Mode) once compilation has had time to land.
- **DX12 unbound-parameter warnings**: can appear on a Simulate shader on DX12 but not Vulkan;
  non-blocking, but consider restructuring as a Screen-Space pass if it's not just noise.
- **PSO precaching**: mentioned as a possible way to pre-warm compilation, not investigated in the
  source material — see Epic's `FPSOPrecacheParams` docs if pursuing this.
- **Fallback chain**: `FindMaterialShaders_RenderThread()`-style helpers should chain through fallback
  Materials on compile failure, ending at the universal Default Material with unwritten pins, rather
  than hard-failing the pass.

Reference: Material output functions in `/Engine/Private/MaterialTemplate.ush`; post-process
boilerplate in `/EGP/ScreenPass/pre.ush` and `post.ush`; a minimal vertex shader example at
`/ExtendedGraphicsProgramming/Shaders/ScreenPass/simple_vs.usf` in the companion demo repo.
