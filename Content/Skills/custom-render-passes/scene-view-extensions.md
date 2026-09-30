---
name: custom-render-passes/scene-view-extensions
description: Scene/SceneRenderer/ViewFamily/View render-thread hierarchy, built-in shader uniform-buffer parameters (View/Scene/SceneTextures), and SceneViewExtension (SVE) architecture for hooking custom passes into the renderer, including per-view persistent state and lifetime gotchas.
---

# Scene, Views & SceneViewExtensions

## Render-thread data hierarchy (outer → inner)

1. **Scene** (`FScene`/`FSceneInterface`) — the render-thread mirror of `UWorld`: render-related
   components (meshes, decals, lights) and their proxies, plus GPU resources including `FGPUScene`
   (GPU-side buffers) and collections like a `DirectionalLights` array.
2. **Scene Renderer** (`FSceneRenderer`) — short-lived, allocated per frame, owns view-families and
   views. Rarely manipulated directly by external code.
3. **View Family** (`FSceneViewFamily`) — one configured viewport rendering to screen or texture.
   - `FSceneViewFamilyContext` — transient/screen-space calculations.
   - `FViewFamilyInfo` — real rendering work.
   - Key fields: `RenderTarget`, `Scene`, `Views`. Created/destroyed freely as locals.
4. **View** (`FSceneView`) — a point of view within a family. A fully renderable view is a `FViewInfo`
   (subclass) plus an `FSceneViewState` (base `FSceneViewStateInterface`). Multiple views per family
   ⇒ cascaded shadows, planar reflections, VR stereo.

Validate before treating a view as fully renderable:
```cpp
if (view.bIsViewInfo) {
    const auto& asViewInfo = static_cast<const FViewInfo&>(view);
}
```

## Built-in shader parameters

Shaders read scene/view state through uniform-buffer parameters on their parameter struct:

| Parameter struct | Contents | Setup |
|---|---|---|
| `FViewUniformShaderParameters` | Camera/transform matrices | `params->View = view.ViewUniformBuffer;` |
| `FSceneUniformParameters` | GPU scene resource data (`FGPUSceneResourceParameters`) | `params->Scene = view.GetSceneUniforms().GetBuffer(graph);` |
| `FSceneTextureShaderParameters` | Scene color/depth/G-buffer (depends on pipeline stage) | `params->SceneTextures = sceneTextures;` |
| `FInstanceCullingDrawParams` | LOD/instancing/Nanite data (nullable) | mesh-material context, see `mesh-material-shaders.md` |

Declare with the matching macro: `SHADER_PARAMETER_STRUCT_REF(FViewUniformShaderParameters, View)`,
`SHADER_PARAMETER_RDG_UNIFORM_BUFFER(FSceneUniformParameters, Scene)`,
`SHADER_PARAMETER_RDG_UNIFORM_BUFFER(FSceneTextureShaderParameters, SceneTextures)`. In HLSL, call
`ResolvedView = ResolveView();` at the top of the shader to handle stereo rendering (VR) correctly.

`View` data layout lives in `VIEW_UNIFORM_BUFFER_MEMBER_TABLE` (`Engine/Public/SceneView.h`).

## SceneViewExtensions (SVE)

SVEs are persistent hooks into the render pipeline for custom passes, split across two threads:

- **Game thread**: a UPROPERTY-bearing object driving per-frame parameter updates
  (`Tick_GameThread()` pushes values to the render-thread copy via `ENQUEUE_RENDER_COMMAND`, per the
  proxy pattern in `threading-and-rdg.md`).
- **Render thread**: the SVE proper, overriding phase-specific virtual callbacks such as
  `PostRenderBasePassDeferred_RenderThread()` (after the deferred base pass, before lighting) and
  `PrePostProcessPass_RenderThread()`. Each phase hands you the relevant render targets and an
  `FRDGBuilder`.

### Manual SVE registration (no framework)

1. SVE constructor takes `const FAutoRegister&` plus your own arguments.
2. Create it **only on the game thread**: `FSceneViewExtensions::NewExtension<FMySVE>(args...)`.
3. Own it as `TSharedPtr<FMySVE, ESPMode::ThreadSafe>` from a World Subsystem (or equivalent) so its
   lifetime is managed.
4. Filter when it actually runs via the `IsActiveThisFrameFunctions` predicate array.

🚨 **Lifetime gotcha:** if the SVE is destroyed alongside its owning subsystem, a render command still
in flight can race against subsystem teardown while reading subsystem parameters. Block subsystem
destruction on a GPU fence until the SVE has actually stopped being used, the same way the proxy
pattern's `FRenderCommandFence` blocks UObject destruction.

### Per-view persistent state

Views and view-families are **recreated every frame** — you cannot index persistent data by pointer.
Use the view's stable key instead: `auto viewID = view.State->GetViewKey();`. There is no explicit
"view destroyed" notification, so persistent-state containers need their own frame-based staleness
cleanup (prune entries not touched this frame/N frames).

### Editor-only warning

SVEs execute **globally**, including during asset-thumbnail generation and Material-preview
rendering — not just the player's actual viewport. Any pass with real side effects (writing shared
state, expensive work) should filter with `IsActiveThisFrameFunctions` on player index, camera actor,
or target render target to avoid firing during previews/thumbnails.

## Data flow: game thread → render thread

Same proxy-update shape as elsewhere:
```cpp
auto* outputPtr = &color_renderThread;
auto updateClearColor = [outputPtr, newClearColor](FRHICommandListImmediate&) {
    *outputPtr = newClearColor;
};
ENQUEUE_RENDER_COMMAND(UpdateClearColor)(MoveTemp(updateClearColor));
```

## Reference

- Renderer-module functions for manual view creation: `CreateAndInitSingleView()`,
  `AllocateViewState()`, obtained via `GetRendererModule()`.
- Header for parameter-struct types referenced above:
  `#include "Runtime/Renderer/Private/SceneRendering.h"`.
- Other struct macros seen in real post-process/mesh-material setups:
  `SHADER_PARAMETER_STRUCT_REF()` (uniform buffer ref), `SHADER_PARAMETER_RDG_UNIFORM_BUFFER()`
  (RDG-bound buffer), `SHADER_PARAMETER_STRUCT_INCLUDE()` (embedded parameter struct).
