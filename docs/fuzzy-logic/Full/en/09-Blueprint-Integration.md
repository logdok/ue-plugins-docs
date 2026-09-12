# 09 — Blueprint Integration

*🇬🇧 English | [🇺🇦 Українська](../uk/09-Blueprint-Integration.md)*

## The Fuzzy Logic Component

`UFuzzyLogicComponent` is available in the **Fuzzy Logic** group and can be added to any actor.

| Property | Value |
|---|---|
| **System Asset** | A shared `UFuzzySystemAsset`; takes priority |
| **Inline System** | A local system, used when no asset is assigned |

Both are writable from Blueprint. The component caches the compiled system and notices when you point it at a different `System Asset`, so swapping behaviour profiles at runtime just works. Editing the `Inline System` struct in place is the one change it cannot see — call **Mark System Dirty** after that.

## Feeding Inputs

| Node | Behavior |
|---|---|
| **Set Input** | Writes a single value; returns `false` if the name isn't among the inputs |
| **Set Inputs** | Writes a `Map<Name,float>`; returns the count of names found |
| **Get Input** | Returns the supplied or default value |
| **Clear Inputs** | Forgets supplied values and returns the system to `Default Value` |

Supplied values persist between evaluations. Only update what has changed.

## Evaluation

| Node | Result |
|---|---|
| **Evaluate** | The full `FFuzzyInferenceResult` |
| **Evaluate Output** | A single number by output name |
| **Evaluate Single Output** | The number of the sole output |
| **Get Last Result** | The previous result without a new computation |

The Evaluate nodes are deliberately not Pure: inference does real work, and a Pure node could silently repeat it for every pin reader.

## Typical Actor Pattern

```text
Event Tick or a timer
  → Set Input (Distance)
  → Set Input (Health)
  → Evaluate Output (Aggression)
  → apply to speed, transition weight, or action selection
```

Use **Evaluate Output** when you need one number. Use **Evaluate** when you also need the degrees, activations, or diagnostics — then read a crisp value out of the result's `Outputs` map with a standard **Find** node.

For a large number of agents, you don't have to evaluate every frame. Evaluate on a timer, on a significant input change, or spread agents across frames.

## Inspection

| Node | Purpose |
|---|---|
| **Is System Valid** | Whether the system compiled without errors |
| **Validate System** | The full list of diagnostics |
| **Get Input Names / Get Output Names** | Names in declaration order |
| **Get Last Output Surface** | The exact surface from the last evaluation |
| **Mark System Dirty** | Reset the cache after editing the `Inline System` struct in place |

**Get Last Output Surface** returns `false` for an unknown output name and before the first `Evaluate`, so a plot can never show a curve the system never produced. A flat zero surface *is* a real answer — "no rule fired" — and comes back as success.

## Shared Systems For Many Agents

A component carries its own copy of the system it evaluates. That is the right trade for a handful of actors and wrong for a crowd: a hundred components pointing at one asset hold a hundred copies of it.

**`UFuzzyLogicSubsystem`** compiles each asset once and shares the result. There is nothing to set up — Unreal creates it with the game instance. Reach it with **Get Game Instance Subsystem**, class **Fuzzy Logic**.

| Node | Purpose |
|---|---|
| **Evaluate Asset** | Full `FFuzzyInferenceResult` for an asset's system |
| **Evaluate Asset Output** | A single number by output name |
| **Prepare Asset** | Compiles ahead of time, e.g. on a loading screen |
| **Validate Asset** | Compiles if needed and returns the diagnostics |
| **Invalidate Asset / Invalidate All** | Drops a cached compilation after changing a system at runtime |
| **Is Asset Prepared** | Whether an asset is already compiled and waiting |
| **Get Cached System Count** | How many systems are currently cached |

```text
Event Tick or a timer
  → Get Game Instance Subsystem (Fuzzy Logic)
  → Make Map (Distance → 450, Health → 72)
  → Evaluate Asset Output (Asset = BehaviorAsset, Output Variable Name = "Aggression")
```

The difference from the component is that **nothing is remembered between calls**: pass every input the system needs each time, because there is no per-actor state to hold them. Variables you leave out fall back to their `Default Value`. There is no `Get Last Result` either.

Use the component for a few actors, per-actor systems, or when you plot a decision. Use the subsystem when many agents share one behaviour asset. Both give identical numbers — the same engine decides.

Editor note: saving a change to a cached asset drops its entry, so a Play-In-Editor session always runs the version you just saved.

## Static Library

`UFuzzyLogicStatics` is convenient for one-off operations without a component:

- `Evaluate Fuzzy System`;
- `Validate Fuzzy System`;
- `Parse Fuzzy Rule` and `Fuzzy Rule To String`;
- `Save/Load Fuzzy System To Json`;
- `Evaluate Fuzzy Set`;
- `Make Fuzzy Term Key`;
- `Describe Fuzzy Set`;
- `Fuzzy Diagnostic To String`.

For repeated evaluation, use the component: it caches the compiled system.

## Result Structure

Break `FFuzzyInferenceResult` to get crisp outputs, input degrees, set and rule activations, and diagnostics. All six fields are Blueprint-readable.

The struct's C++ helpers (`GetOutput`, `GetSingleOutput`, `HasErrors`) are **not** Blueprint nodes. From Blueprint:

- read a crisp value with **Find** on the `Outputs` map;
- check `bSuccess` before applying the result of a system that may come from user-supplied data;
- to distinguish errors from warnings, loop over `Diagnostics` and compare `Severity`, or call **Validate System** on the component.
