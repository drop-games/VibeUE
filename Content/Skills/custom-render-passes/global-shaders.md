---
name: custom-render-passes/global-shaders
description: FGlobalShader fundamentals — shader maps, feature levels, permutations, HLSL file wiring (DECLARE/IMPLEMENT_SHADER_TYPE), the Unreal shader-parser gotchas, all SHADER_PARAMETER_* macro variants, legacy parameter binding, built-in View/Scene parameters, and dispatch via AddDrawScreenPass/ENQUEUE_RENDER_COMMAND.
---

# Global Shaders

The simplest of the three shader classes (`FGlobalShader` < `FMaterialShader` < `FMeshMaterialShader`)
— no Material graph, no geometry, just an `FShader` compiled once per feature level. Good first stop
for a brand-new pass before adding Material or mesh integration.

## Shader maps, feature levels, permutations

A **Shader Map** holds every compiled global shader variant for one feature level (`SM5`, `SM6`,
`ES3_1` as of UE 5.3). Get one via `GetGlobalShaderMap(level)`, or — preferably — from a view:
`TShaderMapRef<FMyShader> myShader{ view.ShaderMap };` (no permutation).

**Permutations** are compile-time variants selected by enum/int/bool template params:
```cpp
class FEnableParallaxMapping : SHADER_PERMUTATION_BOOL("PARALLAX_MAPPING");
class FIterationCount        : SHADER_PERMUTATION_INT("ITER_COUNT", 3);
using FPermutationDomain = TShaderPermutationDomain<FEnableParallaxMapping, FIterationCount>;
```
```cpp
FMyShaderPS::FPermutationDomain permutation;
permutation.Set<FMyShaderPS::FEnableParallaxMapping>(true);
permutation.Set<FMyShaderPS::FIterationCount>(1);
TShaderMapRef<FMyShaderPS> myShader{ view.ShaderMap, permutation };
```
Shadow `RemapPermutation(...)` to collapse redundant permutations you don't want compiled. Convert a
permutation to/from a unique ID with `domain.ToDimensionValueId()`.

## Wiring HLSL to a shader class

1. Shader code lives in `.usf` under the registered shader folder (`setup-and-debugging.md`).
2. In the shader class: `DECLARE_EXPORTED_SHADER_TYPE(FMyClass, Global, )` (or
   `DECLARE_EXPORTED_GLOBAL_SHADER`).
3. In the `.cpp`: `IMPLEMENT_SHADER_TYPE(templateDecl, Class, VirtualShaderPath, MainFunction, Frequency)`,
   e.g. `IMPLEMENT_SHADER_TYPE(, FMyNewVS, TEXT("/Plugin/MyPlugin/myShader.usf"), TEXT("MainVS"), SF_Vertex);`
4. Compilation happens at engine startup; a compile error there **halts startup** (contrast with
   Material shaders, which fail silently later — `material-shaders.md`).

Suffix shader classes by frequency (`VS`, `PS`, `CS`, ...). Keep tightly-paired VS/PS in one `.usf`
with separate entry points (`MainVS`/`MainPS`), branching shared logic on `VERTEXSHADER` /
`PIXELSHADER` / `COMPUTESHADER` preprocessor defines. Give a DLL export token
(e.g. `MYMODULE_API`) only when the shader must be visible outside its own module — otherwise keep it
private to the `.cpp`, encapsulated behind a plain invocation function.

## 🚨 Shader-parser gotchas

The HLSL parser has real, documented bugs:
- Never write the `uniform` keyword explicitly — it's implicit and can crash the parser.
- Never declare multiple uniforms in one statement (`float A, B;`) — only `A` is recognized.
- Never prefix a uniform with an underscore (`_MyParam`) — breaks parsing.
- DX12 reports vertex↔pixel interface mismatches as a generic "invalid pipeline state" error — check
  semantics line up exactly.

## Parameter structs (modern approach — preferred)

```cpp
BEGIN_SHADER_PARAMETER_STRUCT(FMyParams, )
    SHADER_PARAMETER(FMatrix44f, MyMatrix)
END_SHADER_PARAMETER_STRUCT()

class FMyShaderPS : public FGlobalShader
{
    using FParameters = FMyParams;
    SHADER_USE_PARAMETER_STRUCT(FMyShaderPS, FGlobalShader)
};
```

| Kind | Macro |
|---|---|
| Scalar / array | `SHADER_PARAMETER(FMatrix44f, MyMatrix)`, `SHADER_PARAMETER_ARRAY(FUintVector4, MyUvecs, [2])` |
| Texture/buffer SRV (RDG) | `SHADER_PARAMETER_RDG_TEXTURE_SRV(Texture2D<float2>, MyTex)` |
| Texture/buffer SRV (raw RHI) | `SHADER_PARAMETER_SRV(Buffer<uint2>, MyUvecs2)` |
| Read-write UAV (RDG) | `SHADER_PARAMETER_RDG_TEXTURE_UAV(RWTexture2D<float4>, MyOutputTex)` |
| Read-write UAV (raw RHI) | `SHADER_PARAMETER_UAV(RWBuffer<float>, MyOutputFloats)` |
| Sampler | `SHADER_PARAMETER_SAMPLER(SamplerState, MySampler)`, `SHADER_PARAMETER_SAMPLER_ARRAY(SamplerState, MySamplers, [3])` |
| Nested struct | `SHADER_PARAMETER_STRUCT_REF(FViewUniformShaderParameters, View)`, `SHADER_PARAMETER_STRUCT_INCLUDE(...)`, `SHADER_PARAMETER_STRUCT(...)`, `SHADER_PARAMETER_STRUCT_ARRAY(...)` |

Always prefer the `_RDG_` variant for mutable resources (see Critical Rules in `SKILL.md`).

**Uploading**: with plain RHI, `SetShaderParameters(cmds, shaderRef, shaderRef.GetPixelShader(), params)`.
With RDG (preferred): allocate via `graph.AllocParameters<FMyParamStruct>()`, fill fields, call
`SetShaderParameters(...)` inside the `AddPass(...)` lambda — or use a helper like
`AddDrawScreenPass()` which does this for you.

If your parameter struct doesn't enumerate *every* shader uniform (external/legacy bindings mixed in),
use `SHADER_USE_PARAMETER_STRUCT_WITH_LEGACY_BASE(...)` instead of `SHADER_USE_PARAMETER_STRUCT(...)`
to avoid assert failures on the mismatch — this is also the macro Material Shaders use, see
`material-shaders.md`.

## Legacy parameters (older approach, still needed for mesh shaders)

```cpp
LAYOUT_FIELD(FShaderParameter, MyNormalParam);
LAYOUT_FIELD(FShaderResourceParameter, MySrvOrUavOrSamplerParam);
LAYOUT_FIELD(FShaderUniformBufferParameter, MyNestedParamStruct);
```
Bind in the constructor: `MyParam.Bind(initializer.ParameterMap, TEXT("HLSLName"), SPF_Mandatory);`
(use `SPF_Optional` when the uniform might not be referenced by every permutation).

Upload: get scratch memory via `cmds.GetScratchShaderParameters()`, call `SetShaderValue(...)` /
`SetSRVParameter(...)` / `SetUAVParameter(...)`, finalize with `cmds.SetBatchedShaderParameters(...)`.
Cleanup mirrors this via `cmds.GetScratchShaderUnbinds()` + `UnsetSRVParameter(...)` /
`UnsetUAVParameter(...)` + `cmds.SetBatchedShaderUnbinds(...)`. Convention: wrap this in your own
`SetParameters(...)`/`UnsetParameters(...)` member functions. This is the **only** option for
Mesh-Material shaders — see `mesh-material-shaders.md`.

## Built-in View/Scene parameters

Add `SHADER_PARAMETER_STRUCT_REF(FViewUniformShaderParameters, View)` and set
`params->View = view.ViewUniformBuffer;`; add `SHADER_PARAMETER_RDG_UNIFORM_BUFFER(FSceneUniformParameters, Scene)`
and set via `view.GetSceneUniforms().GetBuffer(rdgInstance)`; add
`SHADER_PARAMETER_RDG_UNIFORM_BUFFER(FSceneTextureShaderParameters, SceneTextures)` when hooked at a
pipeline stage that has G-buffer data available. Full detail in `scene-view-extensions.md`.

## Static functions to shadow

- `ShouldCompilePermutation(const FGlobalShaderPermutationParameters&)` — filter compilation by
  platform/API/editor requirements.
- `ModifyCompilationEnvironment(const FShaderPermutationParameters&, FShaderCompilerEnvironment& env)`
  — add virtual paths, `#define` tokens, compiler flags (`ECompilerFlags`).
- `ValidateCompiledResult(EShaderPlatform, const FShaderParameterMap&, TArray<FString>& outErrors)` —
  post-compile validation.

## Dispatch

- **Screen-space**: `AddDrawScreenPass()` — wraps `AddPass(...)`, typically calls
  `SetShaderParameters(...)` for you, and binds the built-in `FScreenPassVS` vertex shader by default.
- **From any thread**: `ENQUEUE_RENDER_COMMAND()`, per `threading-and-rdg.md`.

## Demo reference (Game of Life shaders)

Four global shaders drove the series' demo: **Initialize** (Perlin-style noise via
`GradientNoise3D_ALU()` from `Random.ush`, seeded per-viewport), **Resample** (bilinear downsample on
viewport-resolution change, using `TStaticSamplerState<SF_Bilinear, AM_Clamp, AM_Clamp>::GetRHI()`),
**Simulate** (Conway's rules with smooth transitions, thresholds as plain floats), **Display**
(multiply-blend the sim state onto scene color:
`TStaticBlendState<CW_RGBA, BO_Add, BF_DestColor, BF_Zero>::GetRHI()`, run in
`PrePostProcessPass_RenderThread()`). Sim state ran at **half viewport resolution** — a ~75% shader
cost cut versus full resolution, worth considering for any screen-space simulation texture.
