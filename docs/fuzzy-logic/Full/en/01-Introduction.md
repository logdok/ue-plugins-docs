# 01 — Introduction

*🇬🇧 English | [🇺🇦 Українська](../uk/01-Introduction.md)*

## What Fuzzy Logic Is

An ordinary boolean condition has only two states: true or false. For example, `Distance < 500` instantly flips from `false` to `true` at the 500-unit boundary. For game behavior this is often too abrupt: an agent can flicker between two states when the value hovers near the threshold.

Fuzzy logic describes not just the fact of being "close," but the **degree of membership** in the concept "close." At a distance of 350 it might equal `0.9`, at 600 — `0.45`, and at 1000 — `0`. A single value can partially belong to both the `Near` and `Mid` sets at the same time. This overlap between sets is exactly what creates a smooth transition.

```text
Hard threshold: Distance < 500  →  Attack = true or false
Fuzzy decision: Distance IS Near → Aggression = 0.73
```

The **Fuzzy Logic** plugin implements a Mamdani fuzzy inference system. You describe inputs, outputs, linguistic concepts, and rules, and the system turns current numeric values into smooth numeric decisions.

## A Simple Example

Suppose a drone knows the distance to a reactor and its own integrity:

```text
IF Distance IS Near AND Integrity IS High THEN Aggression IS Aggressive
IF Distance IS Near AND Integrity IS Low  THEN Aggression IS Cautious
IF Distance IS Far                        THEN Aggression IS Watchful
```

The words `Near`, `High`, `Low`, `Aggressive` carry mathematical meaning: each of them is a fuzzy set with a membership function. At runtime the system:

1. computes how much the current inputs belong to each set;
2. determines the firing strength of each rule;
3. combines the rules' consequences into one output curve;
4. converts the curve into a crisp number, e.g. `Aggression = 0.73`.

This process is called **fuzzification → rules → aggregation → defuzzification**.

## Where It's Useful

Fuzzy logic is well suited to problems where several imprecise features should form a single smooth decision:

- combat AI behavior: aggression, caution, distance choice;
- vehicle control: braking, throttle, traction;
- procedural effects: weather intensity, lighting, sound;
- risk, priority, or quality scoring;
- crowd and autonomous-agent behavior;
- blending animations or parameters without abrupt transitions.

A fuzzy system doesn't replace a Behavior Tree, a State Tree, or a scheduler. It answers the question **"how much?"** well and returns a control parameter. Some other system decides how to use that number.

## When A Plain Condition Is Better

Use an exact check when a rule is genuinely discrete: a key was picked up or it wasn't, ammo ran out, an actor was destroyed, network authority belongs to the server. Don't turn something with an unambiguous legal or gameplay boundary into a fuzzy system.

## What the Plugin Provides

- a reusable `UFuzzySystemAsset` and an inline system in the component;
- multiple inputs and multiple outputs in one system;
- nine membership functions and support for custom shapes;
- text rules `IF / THEN`, `AND`, `OR`, `NOT`, `WITH`;
- configurable norms, implication, aggregation, and six defuzzification strategies;
- an editor with live inputs, rule strengths, output numbers, and graphs;
- two-way exchange between Data Asset and JSON;
- complete diagnostic messages;
- Blueprint, C++, and UMG APIs.

Next, go through [02 — Quick Start](02-Quick-Start.md) to build your first system without diving into all the math.
