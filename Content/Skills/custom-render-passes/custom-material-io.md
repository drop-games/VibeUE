---
name: custom-render-passes/custom-material-io
description: Exposing custom inputs (static-global + preprocessor-token pattern) and custom outputs (UMaterialExpressionCustomOutput subclass) to a Material graph for a custom render pass, plus the uncached-output-pin trick for shader-driven loops.
---

# Custom Material Inputs & Outputs

Standard Material pins (Emissive, Base Color, ...) only offer 3 loosely-typed scalar/RGB outputs and
no pass-specific inputs at all — fine for a first pass (`material-shaders.md`), limiting once a pass
needs real data in (e.g. per-cell delta time, neighbor states) or real data out (e.g. multiple typed
channels). This doc covers both extension points; `UMaterialExpressionCustomOutput` is *the* supported
non-engine-modification way to add outputs.

## Custom inputs

**Pattern:**
1. Declare `static` global variable(s) *before* the shader includes the generated Material code.
2. Add a preprocessor token so a Material Function can tell whether those globals actually exist in
   the current shader.
3. Set the globals' values before invoking the Material's output pins.
4. Expose the values to artists via a Material Function containing a custom HLSL node, guarded by
   that same preprocessor token, with stub values for shaders where the globals don't exist.

🚨 **Why the guard token is mandatory, not optional:** "each Material gets compiled against nearly
*every* shader in its domain, including the one that renders the Material preview" — so any shader
using your graph but not defining your globals will still get compiled. Without the guard/stub, that
shader fails to compile or reads garbage.

**Example — Simulate pass custom inputs** (`Simulate.usf`):
```hlsl
static float MaterialDeltaSeconds;
static float2 MaterialNeighborStates[9];
```
The companion Material Function checks `#if defined(GOL_PASS_SIMULATE)` to return the real value or a
stub (return value unused in the stub case, but the function must still return *something* — HLSL
requires it). Exposed values in the series' demo: delta time (for temporal effects), the full 3×3
neighbor-cell-state array, plus derived helpers (neighbor population count, average living-neighbor
state) computed from it.

## Custom outputs

**Base class**: `UMaterialExpressionCustomOutput` — the only extension point available without engine
modification. One output node is scoped to a single shader frequency.

```cpp
UCLASS(CollapseCategories, HideCategories=Object, DisplayName="My Pass Outputs")
class UMaterialExpressionMyPassOutputs : public UMaterialExpressionCustomOutput
{
    UPROPERTY(meta=(RequiredInput=true))
    FExpressionInput RequiredPin;

    UPROPERTY(meta=(RequiredInput=false))
    FExpressionInput OptionalPin;

    UPROPERTY(meta=(OverridingInputProperty=OptionalPin))
    float OptionalPinFallback;   // textbox constant used when OptionalPin isn't wired

    virtual FString GetFunctionName() const override { return TEXT("MyPass_Outputs_"); }
    virtual FString GetDisplayName() const override { return TEXT("My Pass Outputs"); }

#if WITH_EDITOR
    virtual void GetCaption(TArray<FString>& out) const override;
    virtual int32 GetNumOutputs() const override { return 2; }
    virtual EShaderFrequency GetShaderFrequency() override { return SF_Pixel; }
    virtual int32 Compile(FMaterialCompiler* compiler, int32 pinIdx) override;
#endif
};
```

**`Compile()` pattern** — check connection, compile if wired, `INDEX_NONE` if not, always call
`CustomOutput()` (even for an unwired pin):
```cpp
auto doPin = [&](int32 i, FExpressionInput& pin, float* fallback)
{
    if (pinIdx != i) return false;
    if (pin.IsConnected())      codeID = pin.Compile(compiler);
    else if (fallback != nullptr) codeID = compiler->Constant(*fallback);
    else                         codeID = INDEX_NONE;
    return true;
};
// ... call doPin for each pin, then:
compiler->CustomOutput(this, pinIdx, codeID);
```

**Generated HLSL artifacts** for `GetFunctionName() == "MyPass_Outputs_"`:
- Preprocessor tokens `HAVE_MyPass_Outputs__0`, `HAVE_MyPass_Outputs__1`, ...
- Callable functions `MyPass_Outputs__0(matParameters)`, `MyPass_Outputs__1(matParameters)`, ...

**Reading them in the shader**, always behind the `HAVE_*` guard with a sane fallback:
```hlsl
#if HAVE_GoL_Outputs_Init_0
    newState.x = GoL_Outputs_Init_0(matParams);
#else
    newState.x = 0;
#endif
```

## 🚨 Custom output pins are NOT cached — usable for shader-driven loops

"Material code does not cache the results of custom output pins, which means you can invoke the same
pin more than once while changing your input globals to get different results!" This is the mechanism
for a loop where the **shader** owns iteration count/control flow but the **Material graph** owns the
loop body — e.g. the series' six-stage distortion effect (two simple shaders plus a three-stage
simulation shader) leans on this. Not possible with ordinary Material pin evaluation, which is
value-cached per evaluation.

## Multi-output-node pattern (splitting a pass into decision + continuous stages)

A single pass frequency can use more than one `UMaterialExpressionCustomOutput` subclass if the
outputs are logically separate stages — e.g. the series' Simulate pass used one node (3 pins:
underpopulation/resurrection/overpopulation thresholds, `SF_Compute`, with fallback constants via
`OverridingInputProperty`, defaults 2.0/2.5/3.0) for decision logic, and a second node (1 pin,
`SF_Compute`) for the continuous-state update that reads the first stage's result.

## Asset organization

Material Functions go in the plugin's `Content/` folder with **"Expose to Library"** enabled so they
show up in the Material graph's context menu for artists.

## Gotchas

- Always provide a stub/fallback for shader frequencies your node doesn't target — the shader parser
  compiles the Material against every shader in its domain regardless.
- `Compile()`'s pin-index handling must match `GetNumOutputs()`'s declared count exactly.
- Function names must be legal HLSL tokens — avoid leading underscores, same rule as global-shader
  uniforms (`global-shaders.md`).
- Typically one output node per shader frequency your pass uses (Pixel, Compute, ...).
