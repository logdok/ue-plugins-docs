# 10 — Інтеграція на C++

*[🇬🇧 English](../en/10-CPP-Integration.md) | 🇺🇦 Українська*

## Модулі

| Модуль | Призначення |
|---|---|
| `FuzzyLogic` | Типи, рушій висновку, компонент, підсистема, Data Asset, JSON |
| `FuzzyLogicUMG` | Малювання кривих у UMG |
| `FuzzyLogicDemo` | `Runtime`. Актори та панель fire control для двох демо-карт. Ніщо інше від нього не залежить, але це runtime-модуль, і він таки лінкується в запаковану гру — див. [16 — Demo Content](16-Demo-Content.md) |
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

Компонент компілює ліниво й кешує результат. Він порівнює те, з чого було зібрано кеш, тож якщо вказати йому на інший `SystemAsset` — у рантаймі, з Blueprint чи C++, — наступний виклик перекомпілює систему сам.

Єдина зміна, якої він не помічає, — редагування структури `FFuzzySystem` на місці. Після неї викличте `MarkSystemDirty`:

```cpp
FuzzyLogic->InlineSystem.Settings.SampleCount = 401;
FuzzyLogic->MarkSystemDirty();
```

Зміни властивостей у редакторі та `PostLoad` скидають кеш автоматично.

`FFuzzyInferenceEngine::Compile` копіює всю систему всередину рушія, і кожен компонент володіє власним рушієм — тож система, спільна через один `UFuzzySystemAsset`, усе одно дублюється й перекомпільовується для кожного компонента. Для сотень агентів краще поділитися одним скомпільованим рушієм — див. [14 — Architecture And Performance](14-Architecture-And-Performance.md).

## Спільна скомпільована система

`UFuzzyLogicSubsystem` тримає один скомпільований рушій на асет протягом життя game instance. Тоді агенти не мають власної системи — лише свої вхідні значення.

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

із такими членами:

```cpp
UPROPERTY(EditAnywhere, Category = "AI")
TObjectPtr<UFuzzySystemAsset> BehaviorAsset;

TSharedPtr<const FFuzzyInferenceEngine> Brain;
TMap<FName, float> Inputs;
```

Тримайте рушій на весь час життя агента, а не викликайте `GetEngine` щокадру. Пошук дешевий, але тримати — безкоштовно, і це зберігає рушій живим крізь інвалідацію, тож робота, що вже виконується, ніколи не втратить ґрунт під ногами.

| Метод | Призначення |
|---|---|
| `GetEngine` | Спільний скомпільований рушій, компіляція на перший запит. Null для null-асета |
| `EvaluateAsset` / `EvaluateAssetOutput` | Обчислення одним викликом, доступне й у Blueprint |
| `PrepareAsset` | Скомпілювати завчасно, наприклад на екрані завантаження |
| `ValidateAsset` | Скомпілювати за потреби й зібрати діагностику |
| `InvalidateAsset` / `InvalidateAll` | Скинути кеш після зміни системи в рантаймі |
| `IsAssetPrepared`, `GetCachedSystemCount` | Інтроспекція |

Система, що не скомпілювалася, теж кешується, а її діагностика пишеться в лог один раз і повторюється в кожному результаті: щокадрові спроби зібрати зламану систему лише передрукували б ті самі повідомлення.

### Потоки

`GetEngine` може компілювати, тому викликайте його з ігрового потоку. Те, що він повертає, незмінне, а `Evaluate` є `const` — тож будь-яка кількість потоків може обчислювати на одному рушії одночасно:

```cpp
ParallelFor(Agents.Num(), [this](int32 Index)
{
    // Safe: every agent reads the same engine and writes only its own result.
    Agents[Index].Aggression = Brain->Evaluate(Agents[Index].Inputs).GetOutput(TEXT("Aggression"));
});
```

Підготуйте рушій до початку паралельної роботи — `PrepareAsset` на екрані завантаження або `GetEngine` у `BeginPlay`, як вище.

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
