---
name: custom-render-passes/setup-and-debugging
description: Plugin/shader-directory setup for custom render passes, debug console variables, RenderDoc integration, RHI switching, and test-scene recommendations.
---

# Setup & Debugging

## Plugin setup

Custom render passes live in a **Runtime** plugin with `LoadingPhase: PostConfigInit` (shader modules
must be registered before config-driven engine init; Material Shaders additionally *require* this —
see `material-shaders.md`). Private module dependencies: `Projects`, `RenderCore`, `Renderer`,
`RHICore`, `RHI`.

Register the shader source directory so `.usf`/`.ush` files are reachable by virtual path, from both
C++ `#include` and Material Custom-HLSL nodes:

```cpp
#include "Interfaces/IPluginManager.h"

auto myPlugin = IPluginManager::Get().FindPlugin(TEXT("MyPlugin"));
auto myShadersDir = FPaths::Combine(myPlugin->GetBaseDir(), TEXT("Shaders"));
AddShaderSourceDirectoryMapping("/MyShaders", myShadersDir);
```

Then `#include "/MyShaders/myHeader.ush"` works anywhere, including inside a Material Custom node.

## Debug console variables (`Config/ConsoleVariables.ini`)

| CVar | Effect |
|---|---|
| `r.ShaderDevelopmentMode=1` | Enables shader dev features (better errors, hot patterns) |
| `r.ShaderCompiler.EmitWarningsOnLoad=1` | Shader compile warnings on load — very spammy, opt-in only |
| `r.RHICmdBypass=1` | Disables the RHI thread — commands execute synchronously, step-through debuggable |
| `r.RDG.ImmediateMode=1` | Disables the high-level render-thread queuing (RDG/RHI split) |
| `r.RDG.Debug=1` | Enables RDG pipeline validation (catches missing barriers/dependencies) |

> ⚠️ These change threading behavior. **Turn them off again** before trusting that code is actually
> thread-safe — bugs that only appear with real async scheduling won't show up while bypassed.

## RenderDoc

Enable Unreal's built-in RenderDoc plugin, then set for a debuggable capture:
`r.Shaders.Optimize=0`, `r.Shaders.Symbols=1`, `r.Shaders.ExtraData=1`, `r.Shaders.SkipCompression=1`.

## RHI switching

Project Settings → search "RHI" → change → restart editor. Query the active RHI with the console
command `r.RHI.Name`. Different APIs surface different classes of bug — switch when stuck:

- **DX12**: generic "failed to create GraphicsPipeline" for vertex↔pixel shader interface mismatches;
  can hang on some errors.
- **Vulkan**: crashes tend to land in opaque RHI-thread locations, harder to attribute.
- **DX11**: often *masks* problems the other two APIs catch — don't trust "it works on DX11" alone.

For serious debugging: build the engine locally and target **Debug Editor**, not just
**DebugGame Editor** — DebugGame still ships an optimized engine, only game code is debuggable.

## Test scene recommendations

When building a test level for a new pass, include primitive variety that exercises different code
paths:
1. Static meshes with animated materials.
2. Static meshes with multiple materials and deliberately aggressive LODs.
3. Niagara particles (own vertex factory — see `mesh-material-shaders.md`).
4. Skeletal meshes with procedural animation, e.g. `boneRot = rotMax * perlin(elapsedTime*C + uniqueId*D)`,
   to catch bone-transform-dependent bugs in vertex factories.

## Iteration shortcut

**Ctrl+Shift+.** in the editor recompiles global/material shaders without a full engine restart —
the fast loop for HLSL-only changes.
