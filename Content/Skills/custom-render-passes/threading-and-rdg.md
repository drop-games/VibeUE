---
name: custom-render-passes/threading-and-rdg
description: Render-thread threading model, ENQUEUE_RENDER_COMMAND, the proxy pattern for UObject/render-thread decoupling, and the three RHI abstraction layers (global functions, command lists, RDG) with resource/pass management.
---

# Threading & RDG

## Threading model

Three layers:
- **Render Thread (RT)** — top-level render logic runs here, synchronously. This is the primary
  interface a graphics programmer targets.
- **Worker threads** — distribute intermediary work in parallel.
- **RHI thread** — executes raw graphics-API calls; itself multithreaded on modern APIs (Vulkan,
  DX12).

### Queuing work onto the render thread

```cpp
ENQUEUE_RENDER_COMMAND(DebugName)([captured](FRHICommandListImmediate& cmds)
{
    // runs on the RT, possibly a full frame after this call
});
```

🚨 **Capture by copy.** There is frame lag between queueing and execution — local variables and the
UObject that queued the command may already be destroyed/garbage-collected by the time the lambda
runs. Never capture `this` or a reference; copy the plain data you need.

Dereferencing a UObject from a non-game thread is unsafe — but **using a raw pointer purely as a
lookup key** (never dereferenced) is fine; it's just an integer with type metadata at that point.

## Proxy pattern (UObject ↔ render-thread decoupling)

Every renderable object should have a corresponding **proxy struct**: plain data, not a UObject, not
garbage-collected, fully decoupled from its owner.

1. Store the proxy as a private member of the owning UObject/component.
2. Expose a public getter guarded by `check(IsInRenderThread())`.
3. Push updates from the game thread by enqueueing a **copy** of new values via
   `ENQUEUE_RENDER_COMMAND`, then `MoveTemp()`-ing them into the render-thread-only proxy fields
   (avoids extra copies/allocations, especially for `TArray` with inline allocators).
4. On destruction, start an `FRenderCommandFence` in `BeginDestroy()` and wait on it in
   `FinishDestroy()` — this guarantees no queued render command still references the proxy after the
   owning object goes away. (`BeginDestroy`/`FinishDestroy` are separate GC events for exactly this
   kind of async cleanup.)

**Anti-pattern:** a mutex between game and render threads. It defeats the whole async model; only
tolerable for rare, blocking editor-only operations.

### Naming convention

Functions meant to run on a specific thread are often suffixed `_RenderThread` /
`_GameThread` with a matching `check(IsInRenderThread())` / `check(IsInGameThread())` assertion —
but not universally; when the suffix is absent, infer thread context from the parameter types
(`FRHICommandList`, `FRDGBuilder` ⇒ render thread).

## RHI abstraction layers (bottom → top)

### 1. Global RHI functions
`Engine/Source/Runtime/RHI/Public/RHICommandList.h` (footer). Thin wrappers calling virtual methods
on `FDynamicRHI`/`IRHICommandContext`. Mostly prototyping/debug use — e.g. `RHICreateTexture(...)`,
`RHICreateSamplerState()`.

### 2. RHI Command Lists — `FRHICommandList`
A queue of deferred draw calls/state changes, executed asynchronously; multithreaded APIs (DX12,
Vulkan) can distribute the queued commands across worker threads to cut driver overhead.
- `FRHICommandListImmediate` — singleton subtype that executes immediately rather than queueing; this
  is what `ENQUEUE_RENDER_COMMAND` hands you.
- `FRHIComputeCommandList` — compute-only subset.
- Manual construction is rare: `FRHICommandList cmdList; /* queue */ cmdList.FinishRecording();` —
  never touch the list after `FinishRecording()`. Command lists cannot be used from the game thread.
- Global helpers: `DrawClearQuad()` (internal fragment shader for clearing a render target) — read
  RDG's own use of these helpers as the reference pattern.

### 3. Render Dependency Graph — `FRDGBuilder` (preferred for all new code)
A higher-level command list that additionally manages **temporary GPU resource lifetime** and
optimizes the command stream (pooling, reordering, pruning). One frame typically uses only a handful
of `FRDGBuilder` instances — don't create one per small operation.

**Resource management (first half of the graph's life):**
- `graph.Alloc<X>(...)` — allocate C++ objects guaranteed to live until the graph completes.
- `graph.RegisterExternal<X>(...)` — register a resource created outside the graph; idempotent, safe
  even if something else already registered the same resource.
- `graph.Queue<X>Extraction(...)` — promote a temporary graph resource to something that survives
  past graph execution.

**Command organization (second half):**
```cpp
graph.AddPass(RDG_EVENT_NAME("MyPass"), parameters, ERDGPassFlags::...,
    [parameters](FRHICommandListImmediate& cmds) { /* draw/dispatch */ });
```
`AddPass` configures render targets, issues resource barriers, and runs your render-thread lambda; the
shader parameter struct you pass drives automatic dependency tracking.

**Parameter struct RDG integration:**
- `RENDER_TARGET_BINDING_SLOTS()` adds a `RenderTargets` field RDG binds automatically before
  execution.
- Use `SHADER_PARAMETER_RDG_TEXTURE(...)`/`_SRV`/`_UAV` instead of the plain `SHADER_PARAMETER_TEXTURE`
  family for anything mutable — RDG needs to see the dependency to schedule correctly (see
  Critical Rules in `SKILL.md`).

**Global pass helpers** (`Add<X>Pass(...)`): `AddCopyTexturePass(...)`, `AddClearRenderTargetPass(...)`,
`AddSimpleMeshPass(...)`. `AddPass()` itself does **not** upload the parameter struct to shaders — it
lacks shader-binding info. Helpers that accept a shader (e.g. `AddDrawScreenPass()`, see
`global-shaders.md`) handle uploading automatically.

**Manual RDG construction** (rare):
```cpp
FRDGBuilder graph{ FRHICommandListImmediate::Get() };
// ... add passes ...
graph.Execute();
```
Multiple small RDGs reduce pooling opportunities — usually unnecessary; batch work into one graph.

## Minimal RHI-layer example (random render-target clear)

```cpp
auto* rhi = resource->GetTexture2DRHI();
ENQUEUE_RENDER_COMMAND(RandomizedClear)([rhi, clearColor](FRHICommandListImmediate& cmds)
{
    cmds.Transition(FRHITransitionInfo{ rhi, ERHIAccess::Unknown, ERHIAccess::RTV });
    cmds.BeginRenderPass(FRHIRenderPassInfo{ rhi, ERenderTargetActions::DontLoad_Store },
                          TEXT("RandomizedClear"));
    DrawClearQuad(cmds, true, clearColor, false, 0, false, 0);
    cmds.EndRenderPass();
    cmds.Transition(FRHITransitionInfo{ rhi, ERHIAccess::RTV, ERHIAccess::SRVMask });
});
```
`Unknown → RTV` enables render-target writes; `RTV → SRVMask` re-enables shader-resource reads
afterward. (Clearing via RDG's `AddClearRenderTargetPass` is simpler — this demonstrates the raw RHI
layer for cases where you need it.)
