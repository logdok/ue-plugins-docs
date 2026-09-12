# 14 — Architecture And Performance

*🇬🇧 English | [🇺🇦 Українська](../uk/14-Architecture-And-Performance.md)*

## Module Boundaries

`FuzzyLogic` is a runtime module with no Slate or UMG. It contains the data model, the parser, the inference engine, the component, the Data Asset, and JSON.

`FuzzyLogicUMG` depends on the runtime and adds only drawing. `FuzzyLogicEditor` and `FuzzyLogicTests` are `Editor`-type modules and don't end up in the game runtime.

## Compile Once

Rule text is convenient for the author but not for a hot loop. `FFuzzyInferenceEngine::Compile` parses the rules and links names to indices. `Evaluate` works on the compiled form and doesn't re-parse strings.

`UFuzzyLogicComponent` compiles lazily and caches the engine. The cache is reset after loading, an editor change, or `MarkSystemDirty`.

## Computation Cost

The main multipliers:

- the number of agents and the frequency of Evaluate;
- the number of rules and conditions;
- the number of outputs;
- `Sample Count` for surface-based methods.

`Weighted Average` doesn't build a discretized surface and is usually the cheapest. Choose it based on the model's nature, not just for speed.

## Practical Optimization

- A shared `UFuzzySystemAsset` reduces duplication of authored data.
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
