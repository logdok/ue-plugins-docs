# Fuzzy Logic User Guide

*🇬🇧 English | [🇺🇦 Українська](../uk/README.md)*

**Fuzzy Logic** is a Mamdani fuzzy inference plugin for Unreal Engine. It lets you describe behavior with readable `IF / THEN` rules, smoothly blend several conditions, and get one or more numeric decisions without cascades of hard thresholds.

Systems are stored as reusable `UFuzzySystemAsset`s, can run directly inside a `UFuzzyLogicComponent`, are imported and exported as JSON, are checked by the editor, and are visualized in Slate and UMG.

> **Fuzzy Logic 1.0**, for Unreal Engine 5.8. What's new in this release — [Release Notes](Release-Notes.md).

## Quick Start

1. Create a **Fuzzy System** via **Content Browser → Add → Miscellaneous → Data Asset**.
2. Add an input variable `Distance` with range `0…2000` and sets `Near`, `Mid`, `Far`.
3. Add an output `Aggression` with range `0…1` and sets `Cautious`, `Bold`.
4. Write the rules:

   ```text
   IF Distance IS Near THEN Aggression IS Cautious
   IF Distance IS Far THEN Aggression IS Bold
   ```

5. Double-click the Data Asset to open it. On the **Inference** tab, change `Distance` on the left and immediately see the `Aggression` number and graph on the right.
6. Add a **Fuzzy Logic** component to an actor, assign the created `System Asset`, call `Set Input`, then `Evaluate Output`.

## Contents

- [Release Notes](Release-Notes.md) — what's new, what's fixed, and what to watch for when upgrading.
1. [Introduction](01-Introduction.md) — why fuzzy logic is needed and how to read its results.
2. [Quick Start](02-Quick-Start.md) — a minimal working system from Data Asset to Blueprint.
3. [Core Concepts](03-Core-Concepts.md) — variables, sets, rules, results, and surfaces.
4. [Membership Functions](04-Membership-Functions.md) — nine built-in shapes and parameter choices.
5. [Authoring A System](05-Authoring-A-System.md) — building a robust rule base in a Data Asset.
6. [Rule Language](06-Rule-Language.md) — the full `IF / THEN` syntax.
7. [Inference And Settings](07-Inference-And-Settings.md) — norms, implication, aggregation, and defuzzification.
8. [Data Asset Editor](08-Asset-Editor.md) — Details, Variables, Inference, Compile Log, and JSON commands.
9. [Blueprint Integration](09-Blueprint-Integration.md) — the component, the library, and the result structure.
10. [C++ Integration](10-CPP-Integration.md) — modules, the inference engine, and extension.
11. [JSON Presets](11-JSON-Presets.md) — two-way exchange, schema, and API.
12. [Diagnostics And Validation](12-Diagnostics-And-Validation.md) — finding errors in the system and its rules.
13. [UMG Visualization](13-UMG-Visualization.md) — plotting graphs in-game.
14. [Fuzzy Drone Arena Demo](14-Drone-Arena-Demo.md) — a ready-made example with nine agents.
15. [Architecture And Performance](15-Architecture-And-Performance.md) — caching, threading, and compute cost.
16. [FAQ](16-FAQ.md) — short answers to common problems.
17. [Demo Content](17-Demo-Content.md) — maps, a separate module, and excluding Demo from the build.
18. [Fuzzy Turret Defense Fuzzy Glossary](18-Turret-Defense-Fuzzy-Glossary.md) — a glossary of variables, sets, and thresholds for the turret demo.
