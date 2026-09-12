# 09 — Blueprint Integration

*🇬🇧 English | [🇺🇦 Українська](../uk/09-Blueprint-Integration.md)*

## The Fuzzy Logic Component

`UFuzzyLogicComponent` is available in the **Fuzzy Logic** group and can be added to any actor.

| Property | Value |
|---|---|
| **System Asset** | A shared `UFuzzySystemAsset`; takes priority |
| **Inline System** | A local system, used when no asset is assigned |

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
  → Evaluate
  → Get Output (Aggression)
  → apply to speed, transition weight, or action selection
```

For a large number of agents, you don't have to evaluate every frame. Evaluate on a timer, on a significant input change, or spread agents across frames.

## Inspection

| Node | Purpose |
|---|---|
| **Is System Valid** | Whether the system compiled without errors |
| **Validate System** | The full list of diagnostics |
| **Get Input Names / Get Output Names** | Names in declaration order |
| **Get Last Output Surface** | The exact surface from the last evaluation |
| **Mark System Dirty** | Reset the cache after manually changing the system's structure |

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

Expand `FFuzzyInferenceResult` to get crisp outputs, input degrees, set and rule activations, and diagnostics. Check `bSuccess` or `Has Errors` before applying the result of a system that may come from user-supplied data.
