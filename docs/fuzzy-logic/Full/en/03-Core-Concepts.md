# 03 — Core Concepts

*🇬🇧 English | [🇺🇦 Українська](../uk/03-Core-Concepts.md)*

## System

`FFuzzySystem` contains:

- `Inputs` — input variables;
- `Outputs` — output variables;
- `Rules` — rule text in authoring order;
- `Settings` — the mathematical inference parameters.

A system can have multiple outputs. The same inputs and rules can simultaneously drive aggression, speed, and altitude.

## Range

`FFuzzyRange` defines a value domain `[Min, Max]`. `Max` must be greater than `Min`. An input outside the range is clamped to the nearest bound before fuzzification.

The range should match the physical meaning of the quantity: `Health 0…100`, `Temperature -40…120`, `Aggression 0…1`. A range that's too wide compresses the useful part of the curves; one that's too narrow often saturates values at the edges.

## Variable

`FFuzzyVariable` has a name, a range, a default value, and a set of fuzzy sets.

- A name starts with a letter and contains only letters, digits, and `_`.
- Names must be unique among all inputs and outputs.
- `Default Value` is used for an input if code hasn't supplied a value.
- For an output, `Default Value` doesn't participate in inference.

## Fuzzy Set

`FFuzzySet` pairs a name like `Near` with a membership function. The function returns a number `μ(x)` within `[0,1]`:

- `0` — the value doesn't belong to the set at all;
- `1` — it fully belongs;
- an intermediate number — partial membership.

For a single input, several sets can be true at once. For example, a distance of 700 might have `Near = 0.3` and `Mid = 0.7`.

## Rule

A rule links fuzzy statements about inputs to a single output set:

```text
IF Distance IS Near AND Health IS Low THEN Aggression IS Cautious WITH 0.8
```

The premise's result is the rule's **firing strength**. `WITH 0.8` additionally caps its influence.

## Fuzzification

Crisp inputs are converted into degrees of membership. The result is stored in `InputMemberships` with keys of the form `Variable.Set`, e.g. `Distance.Near`.

## Aggregated Surface

Each rule shapes part of the output set. `FFuzzyAggregatedSurface` is a discretized curve obtained after combining all such parts for a single output. Most strategies convert this curve into a number.

## Defuzzification

Defuzzification returns a crisp result from the aggregated surface. The most common method, `Centroid`, computes the center of gravity of the area under the curve.

## Result

`FFuzzyInferenceResult` contains not just `Outputs` but also an explanation of the decision:

| Field | Content |
|---|---|
| `bSuccess` | Whether inference completed without errors |
| `InputMemberships` | Degrees of membership of the inputs |
| `OutputActivations` | Activations of the output sets |
| `Outputs` | Crisp numbers by output name |
| `RuleActivations` | Strength of each authored rule |
| `Diagnostics` | Errors, warnings, and info |

## Data Asset and Inline System

`UFuzzySystemAsset` stores a single system as Unreal content. This is the recommended option when the behavior is used by several actors or edited by a designer.

`UFuzzyLogicComponent` also has an `Inline System`. It's useful for a one-off prototype. If a `System Asset` is assigned, it takes priority over the inline data.

Detailed shape selection is described in [04 — Membership Functions](04-Membership-Functions.md).
