# 10 — C++ Integration

*🇬🇧 English | [🇺🇦 Українська](../uk/10-CPP-Integration.md)*

## Modules

| Module | Purpose |
|---|---|
| `FuzzyLogic` | Types, the inference engine, the component, the Data Asset, JSON |
| `FuzzyLogicUMG` | Curve drawing in UMG |
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

If you modify an `FFuzzySystem` directly in memory, call `MarkSystemDirty`. Editor property changes and `PostLoad` reset the cache automatically.

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
