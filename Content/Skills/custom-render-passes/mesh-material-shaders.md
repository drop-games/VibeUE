---
name: custom-render-passes/mesh-material-shaders
description: Rendering actual scene geometry (static/skeletal meshes, particles) into a custom pass — vertex factories, primitive gathering via component+SceneViewExtension, FMeshPassProcessor, legacy shader-parameter binding (no RDG textures allowed), blend/depth-stencil state, and known limitations.
---

# Mesh-Material Shaders

The top of the shader hierarchy (`FShader` → `FGlobalShader` → `FMaterialShader` →
**`FMeshMaterialShader`**): compiled per-Material *and* per-Vertex-Factory, and the only shader class
that draws actual scene primitives instead of a screen-space quad or a compute dispatch.

## Vertex factories

A vertex factory abstracts how a primitive type generates vertex data: static meshes pull from vertex
buffers, skeletal meshes apply bone transforms in the vertex shader, Niagara particles support both
GPU- and CPU-driven storage with their own buffer layouts, third-party plugins (e.g. voxel landscapes)
ship proprietary factories. Writing a shader against the Vertex Factory abstraction — rather than
against one primitive type's raw data — makes it "completely agnostic to all these data formats."

## Primitive gathering (no built-in scene iteration)

There is no engine API to iterate all primitives (or all visible mesh batches for an `FViewInfo`) that
should feed a custom pass. Two options:
- **Custom component** (recommended for a plugin — this is what the series demonstrates): attach a
  component derived from `U_EGP_RenderPassComponent` to every primitive that should participate.
- **Engine fork**: for a scene-wide pass where performance/static-batching matters, modifying engine
  source is the recommended path — a plugin-based pass cannot get the static-batching benefits
  official engine passes get.

### Component-based gathering, step by step

1. Subclass the component; add per-object UPROPERTYs (`BlueprintReadWrite, EditAnywhere`).
2. Define a POD proxy struct copying the component's fields for render-thread use (proxy pattern,
   `threading-and-rdg.md`).
3. Invoke `EGP_PASS_COMPONENT_SIMPLE_PROXY_IMPL(FMyProxy, expr)`.
4. Override `GetPassType()` to return the owning pass's class.
5. Set the SVE template's component-type and proxy-struct parameters to match
   (`T_EGP_RenderPassSceneViewExtension<...>`, see `scene-view-extensions.md`).
6. From the render thread, call `ForEachComponent_RenderThread(lambda)` on the SVE to iterate every
   marked component this frame.

## 🚨 No RDG parameter-struct textures/buffers

"There is no clear way to use parameter structs in a Mesh-Material shader" — RHI resource handles must
already be valid when the pass is set up, before RDG resolves them at execution time.

**Required approach:** bind resources the **legacy** way.
```cpp
class FGoLMeshPS : public FMeshMaterialShader
{
    LAYOUT_FIELD(FShaderResourceParameter, PreviousStateTex);
    LAYOUT_FIELD(FShaderResourceParameter, PreviousStateSampler);

    FGoLMeshPS(const ShaderMetaType::CompiledShaderInitializerType& init)
        : FMeshMaterialShader(init)
    {
        PreviousStateTex.Bind(init.ParameterMap, TEXT("PreviousStateTex"), SPF_Optional);
        PreviousStateSampler.Bind(init.ParameterMap, TEXT("PreviousStateSampler"), SPF_Optional);
    }
};
```
`SPF_Optional` because a permutation may legitimately not reference the uniform.

**Workaround for RDG dependency tracking**: still define a parameter struct purely so RDG knows about
the dependency for scheduling/barriers — but read the actual resource through the RHI handle in your
custom `FMeshMaterialShaderElementData`, not through the struct.

```cpp
BEGIN_SHADER_PARAMETER_STRUCT(FGoLMeshPassParameters, )
    SHADER_PARAMETER_RDG_TEXTURE_SRV(Texture2D<float2>, PrevState)
    SHADER_PARAMETER_STRUCT_REF(FViewUniformShaderParameters, View)
    SHADER_PARAMETER_RDG_UNIFORM_BUFFER(FSceneUniformParameters, Scene)
    RENDER_TARGET_BINDING_SLOTS()
END_SHADER_PARAMETER_STRUCT()
```

### Custom element data

1. Subclass `FMeshMaterialShaderElementData` with your extra fields.
2. Override the shader's `GetShaderBindings()`, call the parent implementation, then bind your custom
   fields via `FMeshDrawSingleShaderBindings`.
3. Populate an instance during mesh-pass-processor execution:
```cpp
FMyShaderElementData element;
elementData.InitializeMeshMaterialData(ViewIfDynamicMeshCommand, proxy, batch, staticMeshID, false);
element.MyFloat  = deltaSeconds;
element.MyVector = axis;
```

## Mesh Pass Processor

**Base class**: `FMeshPassProcessor`. Responsibilities:
1. Override/shadow `AddMeshBatch()`.
2. Compile shaders against the batch's Material **and** Vertex Factory.
3. Choose draw settings — blend mode, depth usage — via `FMeshPassProcessorRenderState`.
4. Allow Material/batch-level overrides.
5. Build the element-data instance (above).
6. Emit draw calls via `BuildMeshDrawCommands()`.

**Shader-compilation shape:**
```cpp
TMeshProcessorShaders<FGoLMeshVS, FGoLMeshPS> shaderRefs;
FMaterialShaderTypes shaderTypes;
shaderTypes.AddShaderType<FGoLMeshVS>();
shaderTypes.AddShaderType<FGoLMeshPS>();
FMaterialShaders materialShaders;
verify(resource.TryGetShaders(shaderTypes, batch.VertexFactory->GetType(), materialShaders));
materialShaders.TryGetVertexShader(shaderRefs.VertexShader);
materialShaders.TryGetPixelShader(shaderRefs.PixelShader);
```

**Compilation restriction** — Mesh-Material shaders must target `MD_Surface`:
```cpp
static bool ShouldCompilePermutation(const FMeshMaterialShaderPermutationParameters& params)
{
    return params.MaterialParameters.MaterialDomain == MD_Surface
        && FMeshMaterialShader::ShouldCompilePermutation(params);
}
```

**Required includes** (or a linker error on the `BuildMeshDrawCommands()` template):
```cpp
#include "Runtime/Renderer/Public/MeshPassProcessor.inl"
#include "SimpleMeshDrawCommandPass.h"
```

## HLSL structure

```hlsl
#include "/Engine/Private/Common.ush"
#include "/Engine/Generated/Material.ush"
#include "/Engine/Generated/VertexFactory.ush"
```
Vertex shader is largely pass-through, invoking the vertex factory for world-space position/tangent
data and the Material for vertex animation (WPO). Interpolants come from the factory's
`FVertexFactoryInterpolantsVSToPS`. Pixel shader routes custom Material output pins (see
`custom-material-io.md`, conditional on `#if HAVE_..._N`) into the pass's target, using
`OPTIONAL_IsFrontFace` where two-sided handling matters:
```hlsl
FMaterialPixelParameters mParameters = GetMaterialPixelParameters(inputs.VFInterpolants, inputs.Position);
CalcMaterialParameters(mParameters, mInputs, inputs.Position, bIsFrontFace);
```

Other vertex-factory/material accessors used in practice: `GetVertexFactoryIntermediates()`,
`VertexFactoryGetWorldPosition()`, `VertexFactoryGetTangentToLocal()`,
`VertexFactoryGetInterpolantsVSToPS()`, `GetMaterialVertexParameters()`,
`GetMaterialWorldPositionOffset()`.

## Pass execution

```cpp
// dispatch
AddSimpleMeshPass(graph, params, lambda /* builds draw commands */);
```
1. Create one mesh-pass-processor per relevant blend mode, if the pass has multiple.
2. Iterate marked components: `ForEachComponent_RenderThread(...)`.
3. Per component, iterate its batches: `EGP::ForEachBatch(view, primitiveProxy, lambda)`.
4. Call `AddMeshBatch()` on the appropriate processor per batch.

`AddDrawDynamicMeshPass()` is a narrower alternative used specifically for decal rendering — most
custom passes want `AddSimpleMeshPass()`, optionally with an `FInstanceCullingManager`.

**Depth mismatch**: when a pass's own render target is a different resolution than the viewport (e.g.
a half-res simulation texture), resample scene depth first:
`EGP::AddDownsampleDepthPass()` with `EDownsampleDepthFilter::Checkerboard`.

## Blend & depth-stencil state reference

```cpp
// Alpha blend
TStaticBlendState<CW_RGBA, BO_Add, BF_SourceAlpha, BF_InverseSourceAlpha>::GetRHI()
// Additive
TStaticBlendState<CW_RGBA, BO_Add, BF_One, BF_One>::GetRHI()
// Multiply
TStaticBlendState<CW_RGBA, BO_Add, BF_DestColor, BF_Zero>::GetRHI()
// Read-only depth test
TStaticDepthStencilState<false, CF_DepthNearOrEqual>::GetRHI()
```
Loading an existing depth buffer read-only:
```cpp
passParams->RenderTargets.DepthStencil = {
    depthBuffer, ERenderTargetLoadAction::ELoad, FExclusiveDepthStencil::DepthRead_StencilNop
};
```

## Known limitations

- **Landscapes**: cannot reliably iterate landscape components in a custom mesh pass — a landscape can
  have hundreds of components, and iterating them all destroys performance. Unsolved in the source
  material; if a pass needs landscape geometry, budget real investigation time or consider whether an
  engine-side approach is warranted instead.
- **Async compile / RDG binding**: same limitations noted in `material-shaders.md` apply here too —
  custom shaders may run against the Default Material initially, and there's no first-party mechanism
  for RDG-resource binding into a mesh shader (the legacy-binding workaround above is the only path).
- **No official batch-iteration API**: `EGP::ForEachBatch()` is the series' own helper — there's no
  shipped equivalent for "give me all visible mesh batches for this view."
