---
name: custom-render-passes
display_name: Custom Render Passes & Shaders
description: Author custom C++ render passes and shaders without engine modification — RDG/RHI, SceneViewExtension hooks, Global/Material/Mesh-Material shaders, custom Material input/output nodes. Use when the user asks to add a custom render pass, screen-space effect, compute shader, global shader, or mesh-material shader; hook into the scene renderer; or debug RDG/shader-compilation errors. For editing existing material graphs use materials; for diagnosing frame cost use profiling/frame-rate.
vibeue_classes: []
unreal_classes:
  - FRDGBuilder
  - FGlobalShader
  - FMaterialShader
  - FMeshMaterialShader
  - FSceneViewExtensionBase
  - FMeshPassProcessor
  - UMaterialExpressionCustomOutput
keywords:
  - RDG
  - RHI
  - global shader
  - material shader
  - mesh pass
  - SceneViewExtension
  - compute shader
  - HLSL
  - custom material output
  - vertex factory
  - render thread
  - shader compilation
---

# Custom Render Passes & Shaders

Distilled from the 7-part "Advanced Graphics Programming in Unreal" series (medium.com/@manning.w27,
UE 5.3–5.5) and its companion demo repo `heyx3/UnrealExtendedGraphicsProgrammingDemo`
(branches `part/1`…`part/7`). The series builds a screen-space "Game of Life" effect end-to-end,
purely as **plugin C++**, no engine fork — and wires every stage through Material graphs so a tech
artist drives parameters without touching C++.

**What this unlocks:** custom multi-pass effects (compute + screen-space + mesh geometry) with
artist-facing parameters, iterated at Material-graph speed instead of C++-recompile speed. That's the
whole point of the Material-integration work in this skill — don't hard-code parameters in C++ when a
Material pin gets the same result with an order-of-magnitude faster iteration loop.

> 🚨 **Verify before use.** This project is UE 5.8; the source articles target 5.3–5.5. This layer of
> the renderer moves slowly, so class/macro names below are likely still correct, but **do not trust
> them verbatim** — `discover_python_class`/grep the actual 5.8 engine source (or
> `Engine/Source/Runtime/{RenderCore,Renderer,RHI}/...`) for the exact current signature before
> writing code against it. Treat this skill as a map of *where to look and what pattern to use*, not a
> copy-paste API reference.

## Critical Rules

### 🚨 Threading: copy into `ENQUEUE_RENDER_COMMAND`, never touch a live UObject off-thread
`ENQUEUE_RENDER_COMMAND(Name)([captured](FRHICommandListImmediate& cmds){ ... })` queues a lambda
that may run a full frame later — **capture by copy**, never by reference to a game-thread local or a
raw UObject pointer you intend to dereference. The standard pattern is a **proxy struct**: plain data,
not a UObject, not garbage-collected, completely decoupled from its owner. Update it from the game
thread by enqueueing a copy of the new values; read it only from the render thread (guard access with
`IsInRenderThread()`). On destruction, start an `FRenderCommandFence` in `BeginDestroy()` and wait on
it in `FinishDestroy()` so the proxy outlives every render command that touches it. Never put a mutex
between game and render threads to "solve" this — it defeats the async architecture.
See `threading-and-rdg.md`.

### 🚨 Shader parser bugs (silent breakage or compiler crash)
- Never write the explicit `uniform` keyword in HLSL params — it's the default and the parser can
  crash on it.
- Never declare two uniforms in one statement (`float A, B;`) — only `A` gets recognized.
- Never give a uniform a leading underscore (`_MyParam`) — breaks the parser.
- DX12 gives a generic "failed to create GraphicsPipeline" for a vertex↔pixel shader interface
  mismatch; Vulkan crashes inside opaque RHI-thread code; DX11 tends to mask bugs the other two APIs
  catch. When something is inexplicable, **switch RHI** (Project Settings → search "RHI" → restart) or
  check with RenderDoc rather than guessing.

### 🚨 RDG resource macros — use the RDG-aware variant for anything mutable
`SHADER_PARAMETER_RDG_TEXTURE`/`_SRV`/`_UAV` (not the plain `SHADER_PARAMETER_TEXTURE`/`_SRV`/`_UAV`)
so `FRDGBuilder` can see the dependency and schedule/barrier correctly. Prefer RDG (`FRDGBuilder`)
over raw `FRHICommandList` for all new code — it pools/reorders/prunes automatically and manages
temporary GPU resource lifetime for you. See `threading-and-rdg.md`.

### 🚨 Material Shaders must load at `PostConfigInit`, and fail silently if they don't
Global Shaders missing at startup abort with a clear error. **Material** shader-maps missing in a
standalone/packaged build just produce sporadic crashes later — no startup error to point you at the
cause. Set the plugin's `LoadingPhase` to `PostConfigInit` in the `.uplugin`. See `material-shaders.md`.

### 🚨 Mesh-Material shaders cannot use RDG parameter-struct textures/buffers
"There is no clear way to use parameter structs in a Mesh-Material shader" — RHI handles must be valid
when the pass is set up, before RDG has resolved them. Bind mesh-shader resources the **legacy** way
(`LAYOUT_FIELD` + manual `Bind()`/`SetShaderValue()`/`SetSRVParameter()`) via a custom
`FMeshMaterialShaderElementData` subtype, not via `SHADER_PARAMETER_RDG_*`. See
`mesh-material-shaders.md`.

### ⚠️ Custom Material output pins are NOT cached — you can call one twice for a shader-driven loop
Unlike normal Material evaluation, invoking the same `UMaterialExpressionCustomOutput` function twice
in HLSL (with different static globals in between) runs the graph twice and gets two different
answers. This is the trick for shader-controlled iteration where the *shader* owns the loop and the
*Material graph* owns the loop body. See `custom-material-io.md`.

### ⚠️ SceneViewExtensions run everywhere, including thumbnails and Material previews
An SVE with no filtering executes during asset-thumbnail generation and Material-preview rendering,
not just the player's viewport. Gate anything expensive or player-specific with
`IsActiveThisFrameFunctions` (filter by player index / camera actor / render target). See
`scene-view-extensions.md`.

### Debugging toolkit (reach for this before guessing)
- `Config/ConsoleVariables.ini`: `r.ShaderDevelopmentMode=1`,
  `r.ShaderCompiler.EmitWarningsOnLoad=1` (spammy), `r.RHICmdBypass=1` (disable RHI thread — step
  through synchronously), `r.RDG.ImmediateMode=1` (disable high-level render thread queuing),
  `r.RDG.Debug=1` (pipeline validation). **Turn these off again before trusting thread-safety.**
- RenderDoc (built-in plugin): `r.Shaders.Optimize=0`, `r.Shaders.Symbols=1`, `r.Shaders.ExtraData=1`,
  `r.Shaders.SkipCompression=1` for a debuggable capture.
- `r.RHI.Name` console command reports the active RHI.
- **Ctrl+Shift+.** in the editor recompiles global/material shaders without a full restart.
- `CaptureViewport` (`call_tool`, toolset `EditorToolset.EditorAppToolset`, per root CLAUDE.md §6) to
  visually confirm a pass — actually look at the image before claiming success.

## Architecture ladder

**Scene graph (outer → inner):**
`FScene`/`FSceneInterface` (render-thread mirror of `UWorld`; owns proxies + `FGPUScene`) →
`FSceneRenderer` (short-lived, per-frame, rarely touched directly) → `FSceneViewFamily` (one
configured viewport/render-target; `FSceneViewFamilyContext` for transient work,
`FViewFamilyInfo` for real rendering) → `FSceneView`/`FViewInfo` (+ `FSceneViewState`; multiple views
per family = cascaded shadows, planar reflections, VR).

**RHI abstraction (bottom → top, use the highest layer that fits):**
raw RHI global functions (`RHICreateTexture`, prototyping only) → `FRHICommandList` (queued command
list; `FRHICommandListImmediate` executes immediately, handed to you inside
`ENQUEUE_RENDER_COMMAND`) → **`FRDGBuilder`** (preferred for all new code — automatic resource
lifetime + pass optimization).

**Shader class hierarchy:** `FShader` → `FGlobalShader` (compiled once per feature level, no
Material) → `FMaterialShader` (compiled per-Material, screen-space/compute, no geometry) →
`FMeshMaterialShader` (compiled per-Material *and* per-VertexFactory, draws actual scene geometry).
Each step up costs more setup but buys artist control (Material graph) and/or geometry integration
(vertex factory).

## Task Index

| I want to... | Sub-doc |
|---|---|
| Set up shader directories, debug flags, RenderDoc, RHI switching | `setup-and-debugging.md` |
| Queue work onto the render thread safely, understand RDG passes/resources | `threading-and-rdg.md` |
| Hook a custom pass into the renderer (post-base-pass, pre-post-process, etc.), read scene depth/GBuffer, get per-view persistent state | `scene-view-extensions.md` |
| Write a first custom shader with no Material involvement (compute or screen-space) | `global-shaders.md` |
| Let a Material graph drive a screen-space/compute pass's parameters | `material-shaders.md` |
| Expose new inputs (e.g. delta-time, neighbor data) or new outputs (beyond BaseColor/Emissive) to a Material graph for a custom pass | `custom-material-io.md` |
| Render actual scene primitives (static/skeletal mesh, particles) into a custom pass | `mesh-material-shaders.md` |

## Related skills
- **materials** — editing/wiring an *existing* Material graph via MaterialService/MaterialNodeService
  (asset-level authoring). This skill is about the C++ framework a Material graph plugs into; use
  `materials` for the node-wiring side once a custom input/output node exists (see
  `custom-material-io.md`).
- **profiling**, **frame-rate** — once a custom pass runs, use these to find out what it costs
  (CPU-bound vs GPU-bound, Insights traces, worst-frame analysis). This skill does not cover
  performance diagnosis.

## Verification

A tool call succeeding is not proof a pass works. After writing/changing a shader or pass:
1. **Ctrl+Shift+.** to recompile, check the Output Log for compile errors (they can be silent-ish for
   Material shaders — see the PostConfigInit rule above).
2. `r.RDG.Debug=1` to catch resource-dependency/barrier mistakes.
3. RenderDoc capture if the visual result is wrong but compiling — inspect the actual draw/dispatch.
4. `CaptureViewport` (`call_tool`, `EditorToolset.EditorAppToolset`) and look at the resulting image
   against what was asked for, per root CLAUDE.md §6.
