# 05 — Authoring A Fuzzy System

*🇬🇧 English | [🇺🇦 Українська](../uk/05-Authoring-A-System.md)*

## Design from the Decision Backward

First decide on the numeric outputs the game actually needs: `Aggression`, `Brake`, `Altitude`, `Priority`. Then list the measurable inputs that explain those decisions. This helps avoid variables and rules that change nothing.

## Names

A variable or set name starts with a letter and contains letters, digits, or `_`. Rules refer to these names literally. Good names describe meaning:

```text
Distance, ReactorHeat, Integrity
Near, Mid, Far
Cautious, Watchful, Aggressive
```

Don't use spaces in identifiers. Input and output names must be unique within the system.

## Ranges and Default Values

The range should cover all useful values. An input outside the range will be clamped. An input's `Default Value` is used until the component receives an explicit value via `Set Input`.

For normalized decisions, `0…1` is convenient. Physical inputs can stay in their natural units: Unreal centimeters, degrees, percent, or seconds.

## Number of Sets

For a first version, three sets are often enough: low, medium, high. Add more only when they represent a distinct design idea. Too many sets sharply increases the possible number of rules.

## Multiple Outputs

A single system can produce several independent decisions:

```text
IF Distance IS Near THEN Aggression IS Aggressive
IF Distance IS Near THEN OrbitSpeed IS Slow
IF ReactorHeat IS Hot THEN Altitude IS High
```

Each rule has one consequence, but several rules can share the same premise for different outputs.

## Editor Workflow

1. Create inputs and their sets in **Details**.
2. Check range coverage on the **Variables** tab.
3. Add outputs and their sets.
4. Write a minimal set of rules.
5. Go to **Inference**: inputs are visible on the left, results on the right.
6. Check edge values, set-intersection points, and typical scenarios.
7. Review the **Compile Log** and fix errors.
8. Save the Data Asset or export a preset via **Save To JSON**.

## Practical Tips

- Cover the entire working range of an input with at least one set.
- Start with the default Mamdani settings and `Centroid`.
- Start with short rules that have one or two conditions.
- Don't try to recreate a table of exact exceptions. For discrete exceptions, plain code is better.
- If a result is surprising, look not just at the number but also at `InputMemberships`, `RuleActivations`, and the aggregated surface.

The full rule syntax is described in [06 — Rule Language](06-Rule-Language.md).
