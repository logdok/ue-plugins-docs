# 04 — Membership Functions

*🇬🇧 English | [🇺🇦 Українська](../uk/04-Membership-Functions.md)*

A membership function defines a value `μ(x)` within `[0,1]`. Each shape's parameters are edited via named fields, so you don't need to memorize the order of an anonymous array.

## Built-in Shapes

| Shape | Parameters | Use |
|---|---|---|
| **Triangle** | `Left`, `Peak`, `Right` | A universal set with a single maximum. An edge can be made to coincide with the peak for a vertical side. |
| **Trapezoid** | `LeftFoot`, `LeftShoulder`, `RightShoulder`, `RightFoot` | A plateau of full membership; convenient for a "normal range" concept. |
| **Gaussian** | `Center`, `Sigma` | A smooth symmetric shape with no kinks. `Sigma` controls the width. |
| **Bell** | `Center`, `Width`, `Slope` | A generalized bell with a controllable flat top and edge steepness. |
| **Sigmoid** | `Center`, `Slope` | A smooth transition from 0 to 1. A negative `Slope` mirrors the shape. |
| **Ramp** | `Foot`, `Shoulder` | A linear rise or fall with saturation at the edges. |
| **JShape** | `Threshold`, `Steepness` | Zero up to the threshold, then a smooth rise starting from zero. |
| **Singleton** | `Value`, `Tolerance` | A narrow, impulse-like set for a discrete or near-discrete result. |
| **Constant** | `Level` | The same degree of membership across the whole range. |

## Inversion

Every shape has an `Inverted` flag. It replaces the degree with `1 - μ(x)`. This lets a single shape produce a falling J-profile, a notch in a Bell, or an opposite Ramp.

`NOT` in a rule also complements the degree. The difference is where it's authored: `Inverted` changes the set itself everywhere, while `NOT` affects only a specific rule statement.

## How to Choose a Shape

- Start with `Triangle` and `Trapezoid`: they're easy to read and tune.
- Use `Ramp` or `Sigmoid` for "the more, the stronger" concepts.
- Choose `Gaussian` or `Bell` when the derivative needs to be smooth.
- `Singleton` pairs best with `Weighted Average` for fast, near-Sugeno-style systems.
- `Constant` is useful for a constant premise or a fixed alpha-cut.

## Overlap and Coverage

Neighboring sets should generally overlap. An intersection near a degree of `0.5` often gives predictable, smooth blending. The exact value depends on the task.

If there's a gap between sets, no rule may fire. If all sets are too wide, the decision becomes insensitive. Check the shapes in the **Variables** and **Inference** tabs.

## A Custom Shape in C++

Create a `USTRUCT` derived from `FFuzzyMembershipFunction` and implement `EvaluateRaw`:

```cpp
USTRUCT(BlueprintType, DisplayName = "Cosine Lobe")
struct FMyMF_CosineLobe : public FFuzzyMembershipFunction
{
    GENERATED_BODY()

    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Fuzzy Logic")
    float Center = 0.0f;

    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Fuzzy Logic")
    float HalfWidth = 1.0f;

    virtual float EvaluateRaw(float X) const override
    {
        const float T = (X - Center) / FMath::Max(HalfWidth, UE_KINDA_SMALL_NUMBER);
        return FMath::Abs(T) >= 1.0f ? 0.0f
            : 0.5f * (1.0f + FMath::Cos(T * UE_PI));
    }
};
```

The editor, JSON serialization, the type registry, and drawing utilities discover the derived struct via reflection. Override `GetSupport`, `GetRepresentativeValue`, `Validate`, and `ToDisplayString` as needed.
