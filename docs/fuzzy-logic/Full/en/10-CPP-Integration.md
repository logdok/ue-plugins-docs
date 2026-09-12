# 10 — C++ Integration

*🇬🇧 English | [🇺🇦 Українська](../uk/10-CPP-Integration.md)*

## Modules

| Module | Purpose |
|---|---|
| `FuzzyLogic` | Types, the inference engine, the component, the subsystem, the Data Asset, JSON |
| `FuzzyLogicUMG` | Curve drawing in UMG |
| `FuzzyLogicDemo` | `Runtime`. Actors and the fire-control panel used by the two demo maps. Nothing else depends on it, but it is a runtime module and does link into a packaged game — see [16 — Demo Content](16-Demo-Content.md) |
| `FuzzyLogicEditor` | The Data Asset editor; don't add to a game module |
| `FuzzyLogicTests` | Automation tests; don't add to a game module |

In your game module's `.Build.cs`:

```csharp
PublicDependencyModuleNames.AddRange(new[]
{
    "Core",
    "CoreUObject",
    "Engine",
    "FuzzyLogic"
});
```

Add `FuzzyLogicUMG` only if you use `UFuzzyCurveDrawLibrary`.

## The Engine as a Value Type

`FFuzzyInferenceEngine` is not a UObject and can live on the stack, in a subsystem, or in a plain struct:

```cpp
#include "Inference/FuzzyInferenceEngine.h"

FFuzzyInferenceEngine Engine;
if (!Engine.Compile(System))
{
    for (const FFuzzyDiagnostic& Diagnostic : Engine.GetCompileDiagnostics())
    {
        UE_LOG(LogTemp, Warning, TEXT("%s"), *Diagnostic.ToString());
    }
}

const TMap<FName, float> Inputs = {
    { TEXT("Distance"), 450.0f },
    { TEXT("Health"), 72.0f }
};
const FFuzzyInferenceResult Result = Engine.Evaluate(Inputs);
const float Aggression = Result.GetOutput(TEXT("Aggression"), 0.0f);
```

`Compile` parses the rules and links them to indices once. After that, `Evaluate` no longer looks up rule strings.

## Building a System in Code

```cpp
#include "Types/FuzzyMembershipFunction.h"

FFuzzySystem System;

FFuzzyVariable Distance(TEXT("Distance"), FFuzzyRange(0.0f, 2000.0f));
Distance.DefaultValue = 1000.0f;
Distance.AddSet(FFuzzySet::Make<FFuzzyMF_Trapezoid>(
    TEXT("Near"), 0.0f, 0.0f, 300.0f, 700.0f));
Distance.AddSet(FFuzzySet::Make<FFuzzyMF_Trapezoid>(
    TEXT("Far"), 500.0f, 900.0f, 2000.0f, 2000.0f));
System.AddInput(MoveTemp(Distance));

FFuzzyVariable Aggression(TEXT("Aggression"), FFuzzyRange(0.0f, 1.0f));
Aggression.AddSet(FFuzzySet::Make<FFuzzyMF_Triangle>(
    TEXT("Cautious"), 0.0f, 0.0f, 0.6f));
Aggression.AddSet(FFuzzySet::Make<FFuzzyMF_Triangle>(
    TEXT("Bold"), 0.4f, 1.0f, 1.0f));
System.AddOutput(MoveTemp(Aggression));

System.AddRule(TEXT("IF Distance IS Near THEN Aggression IS Cautious"));
System.AddRule(TEXT("IF Distance IS Far THEN Aggression IS Bold"));
```

## Component

```cpp
#include "FuzzyLogicComponent.h"

FuzzyLogic->SystemAsset = BehaviorAsset;
FuzzyLogic->SetInput(TEXT("Distance"), DistanceToTarget);
const FFuzzyInferenceResult Result = FuzzyLogic->Evaluate();
```

The component compiles lazily and caches the result. It compares what the cache was built from, so pointing it at a different `SystemAsset` — at runtime, from Blueprint or C++ — recompiles on the next call by itself.

Editing the `FFuzzySystem` struct in place is the one change it cannot detect. Call `MarkSystemDirty` after that:

```cpp
FuzzyLogic->InlineSystem.Settings.SampleCount = 401;
FuzzyLogic->MarkSystemDirty();
```

Editor property changes and `PostLoad` reset the cache automatically.

`FFuzzyInferenceEngine::Compile` copies the whole system into the engine, and every component owns its own engine, so a system shared through one `UFuzzySystemAsset` is still duplicated and re-compiled per component. For hundreds of agents, share one compiled engine instead — see [14 — Architecture And Performance](14-Architecture-And-Performance.md#memory-per-agent).

## Sharing One Compiled System

`UFuzzyLogicSubsystem` keeps one compiled engine per asset for the lifetime of the game instance. Agents then hold no system of their own — only their own input values.

```cpp
#include "FuzzyLogicSubsystem.h"

void AMyAgent::BeginPlay()
{
    Super::BeginPlay();

    if (UGameInstance* GameInstance = GetGameInstance())
    {
        // Compiled on the first agent that asks; every agent after that gets the same engine.
        Brain = GameInstance->GetSubsystem<UFuzzyLogicSubsystem>()->GetEngine(BehaviorAsset);
    }
}

void AMyAgent::Tick(float DeltaSeconds)
{
    Super::Tick(DeltaSeconds);

    if (!Brain.IsValid())
    {
        return;
    }

    Inputs.Add(TEXT("Distance"), DistanceToTarget);
    Inputs.Add(TEXT("Health"), CurrentHealth);

    Aggression = Brain->Evaluate(Inputs).GetOutput(TEXT("Aggression"), 0.5f);
}
```

with these members:

```cpp
UPROPERTY(EditAnywhere, Category = "AI")
TObjectPtr<UFuzzySystemAsset> BehaviorAsset;

TSharedPtr<const FFuzzyInferenceEngine> Brain;
TMap<FName, float> Inputs;
```

Hold the engine for the agent's lifetime rather than calling `GetEngine` every frame. A lookup is cheap, but holding it is free — and it keeps that engine alive across an invalidation, so work already in flight never loses the ground under it.

| Method | Purpose |
|---|---|
| `GetEngine` | The shared compiled engine, compiling on first request. Null for a null asset |
| `EvaluateAsset` / `EvaluateAssetOutput` | One-call evaluation, also available to Blueprint |
| `PrepareAsset` | Compile ahead of time, e.g. during a loading screen |
| `ValidateAsset` | Compile if needed and collect the diagnostics |
| `InvalidateAsset` / `InvalidateAll` | Drop a cached compilation after changing a system at runtime |
| `IsAssetPrepared`, `GetCachedSystemCount` | Introspection |

A system that fails to compile is cached too, and its diagnostics are logged once and repeated in every result — retrying a broken system every frame would only reprint the same messages.

### Threading

`GetEngine` may compile, so call it from the game thread. What it returns is immutable and `Evaluate` is `const`, so any number of threads may evaluate the same engine at once:

```cpp
ParallelFor(Agents.Num(), [this](int32 Index)
{
    // Safe: every agent reads the same engine and writes only its own result.
    Agents[Index].Aggression = Brain->Evaluate(Agents[Index].Inputs).GetOutput(TEXT("Aggression"));
});
```

Prepare the engine before the parallel work starts — `PrepareAsset` on a loading screen, or `GetEngine` in `BeginPlay` as above.

## Surface and Defuzzifiers

```cpp
FFuzzyAggregatedSurface Surface;
if (Engine.BuildOutputSurface(TEXT("Aggression"), Result.RuleActivations, Surface))
{
    const float Center = FuzzyLogic::Defuzzify::Centroid(Surface);
}
```

`Centroid`, `Bisector`, `MeanOfMaxima`, `SmallestOfMaxima`, `LargestOfMaxima`, `WeightedAverage`, and a generic `Apply` are all available.

## Threading

After `Compile`, the `Evaluate` method is `const`, so an immutable compiled engine can be used for parallel evaluations. Don't call `Compile` concurrently with `Evaluate` on the same instance. The UObject component must be used according to Unreal's normal threading rules.
