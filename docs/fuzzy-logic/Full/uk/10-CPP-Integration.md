# 10 — Інтеграція на C++

*[🇬🇧 English](../en/10-CPP-Integration.md) | 🇺🇦 Українська*

## Модулі

| Модуль | Призначення |
|---|---|
| `FuzzyLogic` | Типи, рушій висновку, компонент, Data Asset, JSON |
| `FuzzyLogicUMG` | Малювання кривих у UMG |
| `FuzzyLogicEditor` | Редактор Data Asset; не додавайте до game-модуля |
| `FuzzyLogicTests` | Automation-тести; не додавайте до game-модуля |

У `.Build.cs` модуля гри:

```csharp
PublicDependencyModuleNames.AddRange(new[]
{
    "Core",
    "CoreUObject",
    "Engine",
    "FuzzyLogic"
});
```

Додайте `FuzzyLogicUMG`, лише якщо використовуєте `UFuzzyCurveDrawLibrary`.

## Рушій як value type

`FFuzzyInferenceEngine` не є UObject і може жити на стеку, у підсистемі або звичайній структурі:

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

`Compile` парсить і зв’язує правила з індексами один раз. `Evaluate` після цього не шукає рядки правил.

## Побудова системи кодом

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

## Компонент

```cpp
#include "FuzzyLogicComponent.h"

FuzzyLogic->SystemAsset = BehaviorAsset;
FuzzyLogic->SetInput(TEXT("Distance"), DistanceToTarget);
const FFuzzyInferenceResult Result = FuzzyLogic->Evaluate();
```

Якщо ви змінюєте `FFuzzySystem` у пам’яті напряму, викличте `MarkSystemDirty`. Заміна властивостей у редакторі та `PostLoad` скидають кеш автоматично.

## Поверхня й дефазифікатори

```cpp
FFuzzyAggregatedSurface Surface;
if (Engine.BuildOutputSurface(TEXT("Aggression"), Result.RuleActivations, Surface))
{
    const float Center = FuzzyLogic::Defuzzify::Centroid(Surface);
}
```

Доступні `Centroid`, `Bisector`, `MeanOfMaxima`, `SmallestOfMaxima`, `LargestOfMaxima`, `WeightedAverage` та універсальний `Apply`.

## Потоки

Після `Compile` метод `Evaluate` є `const`, тому незмінний скомпільований engine можна використовувати для паралельних оцінювань. Не викликайте `Compile` одночасно з `Evaluate` для того самого екземпляра. UObject-компонент має використовуватися відповідно до правил потоків Unreal.
