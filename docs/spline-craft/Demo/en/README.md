# SplineCraft Demo User Guide

*🇬🇧 English | [🇺🇦 Українська](../uk/README.md)*

> ⚠️ **This is the SplineCraft Demo (evaluation) edition.** Every feature works exactly like the full plugin; only *how much* you can build with it is capped, an on-screen watermark is shown, and it does nothing in Shipping builds. Full list: [Demo Limitations](00-Demo-Limitations.md).

SplineCraft Demo is a tool for quickly generating structures (compositions of `StaticMesh` and
`Actor` objects) along a spline: fences, railings, balustrades, colonnades, arcades,
barricades, bridges, roads and more. Everything is configured on a single `ADemoSplineCraftActor`
through the **Details** panel (category **SplineCraft Demo Configuration**), and the geometry
rebuilds automatically after every property change.

> **SplineCraft Demo 7.0.0**, for Unreal Engine 5.8. What's new in this release — [Release Notes](Release-Notes.md).

## Quick Start

1. **Install the plugin** — from Fab / Epic Games Launcher (*Install to Engine*), or by
   copying the `SplineCraftDemo` folder into `<Project>/Plugins/`.
2. **Enable it** — *Edit → Plugins → SplineCraft Demo → Enabled*, restart the editor.
3. **Drop a SplineCraft Demo Actor** into the level (from the *Place Actors* browser).
4. **Shape the spline** — move the points in the viewport, or open
   **SplineCraft Demo Configuration → Utils → Alignment Spline Points**, pick a **Tool**
   (e.g. `Rectangle` or `Ellipse`) and set its options.
5. **Assign meshes** — in the **POSTS** section add an entry and assign a post **Static Mesh**;
   in the **SECTIONS** section, a panel **Static Mesh**. The geometry builds immediately.

For the detailed walk-through see [01. Quick Start](01-Quick-Start.md). Ready-made styles live
in `Content/Samples` (`DA_Preset_IronFence`); assign a preset to the **Preset** field and press
**Apply Preset**.

## Element vocabulary

- **Posts** — a mesh at each spline point (fence post, baluster).
- **Sections** / **Tubes** — a mesh spanning the gap between consecutive points (panel,
  railing, rail).
- **Knobs** — decorative meshes on a post or an element at a weighted position.
- **Free Knobs** — knobs in the actor's local space, not bound to spline points.
- **Polygons** — filled shapes generated with a procedural mesh.

## Contents

| # | Chapter | What's inside |
|---|---------|---------------|
| 00 | [Demo Limitations](00-Demo-Limitations.md) | What the evaluation edition caps, and what is unchanged |
| – | [Release Notes](Release-Notes.md) | What each release adds, fixes and what to watch for |
| 01 | [Quick Start](01-Quick-Start.md) | Install, enable, a first structure in 5 steps |
| 02 | [Spline Points and Alignment](02-Spline-Points-And-Alignment.md) | Manual editing, 13 alignment tools, non-sticky tools |
| 03 | [Posts](03-Posts.md) | A mesh at each point; alternate / special / random meshes |
| 04 | [Sections & Tubes](04-Sections-And-Tubes.md) | `FDemoSCElement`, filling the gap, paddings, knobs on the element |
| 05 | [Knobs and Free Knobs](05-Knobs-And-Free-Knobs.md) | Three kinds of knob; `Mesh` vs `Actor` mode |
| 06 | [Polygons](06-Polygons.md) | Filled shapes, `Follow Curve`, concave outlines, collision |
| 07 | [Visibility Rules and Openings](07-Visibility-Rules.md) | `FDemoSCVisibleConfig` by index; `OPENINGS` by distance |
| 08 | [Randomization and Random Seed](08-Randomization.md) | Spread of scale / shift / rotation; a reproducible seed |
| 09 | [Line vs Curve Mode](09-Line-vs-Curve-Mode.md) | ISM vs Spline Mesh; global and per-element mode |
| 10 | [Runtime, collision, events, presets, scenarios](10-Runtime-And-Blueprint-API.md) | `UpdateAfterChangeAnyProperty`, Stats, `Collision`, hit events, presets, Build Scenario |
| 11 | [Merge to Static Mesh](11-Merge-To-Static-Mesh.md) | `Merge to Static Mesh`, `Replace Spline Actor`, Nanite |
| 12 | [Additional Actors](12-Additional-Actors.md) | `FDemoSCAdditionalActor`, spawned in `BeginPlay` |
| 13 | [Performance and Baking](13-Performance-And-Baking.md) | What a structure costs; build and bake |
| 14 | [FAQ](14-FAQ.md) | Short answers to common questions |

## Requirements

- Unreal Engine **5.8**

## Support

- Discord: https://discord.gg/BQ69zYGy
