# 14 — Architecture And Performance

*🇬🇧 English | [🇺🇦 Українська](../uk/14-Architecture-And-Performance.md)*

## Module Boundaries

`FuzzyLogic` is a runtime module with no Slate or UMG. It contains the data model, the parser, the inference engine, the component, the subsystem, the Data Asset, and JSON.

`FuzzyLogicUMG` depends on the runtime and adds only drawing. `FuzzyLogicEditor` and `FuzzyLogicTests` are `Editor`-type modules and don't end up in the game runtime.

`FuzzyLogicDemo` is a `Runtime` module holding the demo maps' actors and the fire-control panel. The core doesn't depend on it, but being a runtime module it *does* link into a packaged game, and its actor CDOs hard-reference `/Engine/BasicShapes` meshes. See [16 — Demo Content](16-Demo-Content.md) for how to leave it out.

## Compile Once

Rule text is convenient for the author but not for a hot loop. `FFuzzyInferenceEngine::Compile` parses the rules and links names to indices. `Evaluate` works on the compiled form and doesn't re-parse strings.

`UFuzzyLogicComponent` compiles lazily and caches the engine. The cache is reset after loading, an editor change, or `MarkSystemDirty`.

## Computation Cost

The main multipliers:

- the number of agents and the frequency of Evaluate;
- the number of rules and conditions;
- the number of outputs;
- `Sample Count` for surface-based methods.

`Weighted Average` doesn't build a discretized surface and is usually the cheapest. Choose it based on the model's nature, not just for speed. The one exception is a negated consequent (`THEN X IS NOT A`): a complemented shape has no analytic representative value, so the engine integrates it numerically — that rule stops being cheap.

## Memory Per Agent

### What actually gets duplicated

It is easy to assume that pointing a hundred actors at one `UFuzzySystemAsset` means one system in memory. That is true of the *authored* data — the asset itself exists once — but not of the runtime form.

When a component evaluates for the first time, it calls `FFuzzyInferenceEngine::Compile`, and `Compile` takes a **full copy** of the `FFuzzySystem`: every variable, every set with its membership-function shape, every rule string, the settings. Each `UFuzzyLogicComponent` owns its own `FFuzzyInferenceEngine`, so:

```text
1 asset  ×  100 components  =  100 copies of the system  +  100 separate compilations
```

Each component also keeps its last `FFuzzyInferenceResult` alive between calls — three maps and two arrays — so it can answer `Get Last Result` and `Get Last Output Surface`.

For a handful of actors none of this matters and the component is the right tool. It becomes worth thinking about somewhere in the hundreds, or when the same system is evaluated every frame by many agents at once.

### Sharing one compiled engine

The plugin ships the answer to this: **`UFuzzyLogicSubsystem`**, a game instance subsystem that compiles each asset once and hands the compiled form to everyone who asks. A hundred agents on one behaviour asset then share one copy of the system and one compilation, and each keeps only its own input values.

No setup is needed — Unreal creates the subsystem with the game instance. Full usage is in [09 — Blueprint Integration](09-Blueprint-Integration.md#shared-systems-for-many-agents) and [10 — C++ Integration](10-CPP-Integration.md#sharing-one-compiled-system).

The short version, in Blueprint:

```text
Get Game Instance Subsystem (Fuzzy Logic)
  → Evaluate Asset Output (Asset = BehaviorAsset, Inputs, Output Variable Name = "Aggression")
```

and in C++, holding the engine for the agent's lifetime:

```cpp
void AMyAgent::BeginPlay()
{
    Super::BeginPlay();

    if (UGameInstance* GameInstance = GetGameInstance())
    {
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

    // One small map per agent instead of one whole system per agent.
    Inputs.Add(TEXT("Distance"), DistanceToTarget);
    Inputs.Add(TEXT("Health"), CurrentHealth);

    Aggression = Brain->Evaluate(Inputs).GetOutput(TEXT("Aggression"), 0.5f);
}
```

### Component or subsystem?

Neither is a replacement for the other.

| | `UFuzzyLogicComponent` | `UFuzzyLogicSubsystem` |
|---|---|---|
| Memory | One copy of the system per actor | One copy per asset, whole game |
| Compilations | One per actor | One per asset |
| Inputs | Remembered between calls; update only what changed | Supplied fresh on every call |
| Last result | `Get Last Result`, `Get Last Output Surface` | Not kept — hold it yourself |
| Setup | Add a component, assign an asset | Nothing |
| Per-actor tuning | Each actor may carry its own `Inline System` | One shared system, no per-actor variation |
| Threading | Game thread | `Evaluate` may run on worker threads |

The component is the better default, and stays the right answer for a handful of actors, for anything that wants per-actor systems, and for anything that plots its own decision. Reach for the subsystem when many agents share one behaviour, or when evaluation needs to run off the game thread.

Measure before switching. A dozen agents evaluating on a timer will never show this in a profile.

## Practical Optimization

- A shared `UFuzzySystemAsset` reduces duplication of authored data (see the note on runtime memory above).
- Don't call the one-off `Evaluate Fuzzy System` every frame: it compiles the system on every call.
- For repeated decisions, use the component or a stored `FFuzzyInferenceEngine`.
- Evaluate slow AI decisions on a timer, not necessarily in Tick.
- Update only the inputs that changed.
- Reduce `Sample Count` only after measuring error and performance.

## Thread Safety

Separate `FFuzzyInferenceEngine` instances are independent. An engine that's immutable after compilation allows parallel `Evaluate` calls, since the method is `const`. Don't compile the same instance concurrently with evaluating it. The UObject and ActorComponent are subject to Unreal's normal Game Thread rules.

## Data and Packaging

The runtime doesn't need external JSON if the system is stored in a Data Asset. `FuzzyLogicEditor` and `FuzzyLogicTests` aren't loaded in a shipping build. `FuzzyLogicUMG` is only needed by projects with runtime graphs.

## Extensibility

New membership functions are added as `USTRUCT`s derived from `FFuzzyMembershipFunction`. The registry, JSON, and drawing all use reflection, so a central switch for each new type isn't needed.
