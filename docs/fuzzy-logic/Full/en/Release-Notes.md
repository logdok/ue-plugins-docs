# Release Notes

*🇬🇧 English | [🇺🇦 Українська](../uk/Release-Notes.md)*

A short summary of each release — what is new, what is fixed, and anything worth knowing
before you upgrade. Newest first.

---

## 1.0

The first public release. **Unreal Engine 5.8.**

### What's in it

**A complete Mamdani fuzzy inference engine.** Author fuzzy systems as reusable
`UFuzzySystemAsset` data assets or inline on a `UFuzzyLogicComponent`, with readable
`IF / THEN` rules (`AND`, `OR`, `NOT`, `WITH`) instead of cascades of hard thresholds. See
[03 — Core Concepts](03-Core-Concepts.md) and [06 — Rule Language](06-Rule-Language.md).

**Nine built-in membership-function shapes**, plus support for custom shapes authored in C++
as a `USTRUCT` derived from `FFuzzyMembershipFunction` — discovered automatically by the
editor, JSON serialization, and drawing code through reflection, no central switch to update.
See [04 — Membership Functions](04-Membership-Functions.md).

**Full control over the inference math.** Selectable T-norms and S-norms for `AND`/`OR`,
`Clip` or `Scale` implication, three aggregation strategies, and six defuzzification methods
(`Centroid`, `Bisector`, `Mean/Smallest/Largest of Maxima`, `Weighted Average`). See
[07 — Inference And Settings](07-Inference-And-Settings.md).

**A live asset editor.** The Data Asset editor's Inference tab drives every input from
sliders while showing live output numbers, aggregated curves, and rule firing strengths side
by side — plus a Compile Log with per-rule diagnostics. See
[08 — Data Asset Editor](08-Asset-Editor.md).

**Two-way JSON exchange.** Export a system to a portable, versioned JSON preset and reimport
it elsewhere — validated end to end before anything on disk changes, in the editor, in
Blueprint, or in C++. See [11 — JSON Presets](11-JSON-Presets.md).

**Multiple outputs per system**, each independently ruled and defuzzified, so one system can
drive aggression, speed, and altitude at once. See
[05 — Authoring A System](05-Authoring-A-System.md).

**Full Blueprint and C++ surfaces.** `UFuzzyLogicComponent` and `UFuzzyLogicStatics` cover
setting inputs, evaluating, and inspecting results from Blueprint; `FFuzzyInferenceEngine` is
a plain value type usable on the stack or in a subsystem from C++. See
[09 — Blueprint Integration](09-Blueprint-Integration.md) and
[10 — C++ Integration](10-CPP-Integration.md).

**UMG plotting helpers.** `FuzzyLogicUMG` draws membership functions, aggregated surfaces, and
value markers straight into a widget's `On Paint`, using the exact curve the engine used to
decide. See [13 — UMG Visualization](13-UMG-Visualization.md).

**Two runnable demo scenes.** Fuzzy Drone Arena (nine agents sharing one system) and Fuzzy
Turret Defense (three chained Mamdani systems driving a turret's threat assessment, tracking,
and fire control) ship in the plugin and can be excluded from a packaged build. See
[16 — Demo Content](16-Demo-Content.md),
[17 — Fuzzy Drone Arena Demo](17-Drone-Arena-Demo.md), and
[18 — Fuzzy Turret Defense Fuzzy Glossary](18-Turret-Defense-Fuzzy-Glossary.md).

### Upgrading

Nothing to upgrade from — this is the first release. Start at the
[Quick Start](README.md#quick-start).

---

The plugin's own `CHANGELOG.md` is not shipped separately; this page is the record of what each
release contains.
