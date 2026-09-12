# 16 — FAQ

*🇬🇧 English | [🇺🇦 Українська](../uk/16-FAQ.md)*

## Where should I start if I don't know fuzzy logic?

Read [01 — Introduction](01-Introduction.md), then work through [02 — Quick Start](02-Quick-Start.md). For a first system, Triangle, Trapezoid, default settings, and Centroid are enough.

## How is Fuzzy Logic different from a Behavior Tree?

Fuzzy Logic computes smooth degrees and numeric decisions. A Behavior Tree organizes task sequencing and selection. They combine well: a fuzzy result becomes a score, a weight, or a task parameter.

## Why is the output always the midpoint of the range?

Likely no rule fired, or no rule writes to that output. Check the Compile Log, the coverage of the input sets, and `RuleActivations`.

## Why does Set Input return false?

There's no input with that name in the currently active system. Check the `System Asset`, the name's case, and `Get Input Names`.

## Do I need to set all inputs before every Evaluate?

No. The component stores supplied values. Missing ones use `Default Value`. `Clear Inputs` resets all inputs to their defaults.

## Can I have multiple outputs?

Yes. Use `Evaluate` to get the whole map, or `Evaluate Output` by name. `Evaluate Single Output` is meant only for a system with one output.

## Why aren't the editor's results saved?

The Inference sliders are a temporary preview. Change the `Default Value` in Details if it should be the asset's starting value.

## How do I import JSON into a Data Asset?

Open the asset and click **Load Preset**. The import is validated before the data changes and supports Undo. The reverse direction is **Save To JSON**.

## Is JSON needed in a packaged game?

No, if the component uses a Data Asset. JSON is only needed when your game deliberately loads external data through the API.

## Why is a rule marked Invalid — skipped?

The parser couldn't link it to the system. Hover over it or open the Compile Log: the message will point to an unknown token, variable, set, or a mixed operator.

## Which defuzzification method should I choose?

Start with Centroid. Weighted Average suits fast systems with Singleton or symmetric outputs. The maximum-based methods are needed when the most-supported region matters more than a smooth balance across the whole curve.

## Can I add a custom membership function?

Yes. Derive a `USTRUCT` from `FFuzzyMembershipFunction` and implement `EvaluateRaw`. A detailed example is in [04 — Membership Functions](04-Membership-Functions.md).

## Can the engine be used from multiple threads?

After compilation, an immutable `FFuzzyInferenceEngine` can be evaluated in parallel. Don't call `Compile` concurrently with `Evaluate` on the same instance. The component remains a UObject and typically runs on the Game Thread.

## Where can I see a full working example?

Open `/FuzzyLogic/Demo/FuzzyDroneArena/Maps/FuzzyDroneArena` or `/FuzzyLogic/Demo/FuzzyTurretDefense/Maps/FuzzyTurretDefense`. The structure and how to disable the demo are described in [17 — Demo Content](17-Demo-Content.md).
