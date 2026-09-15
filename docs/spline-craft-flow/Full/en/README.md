# SplineCraft Flow User Guide

*🇬🇧 English | [🇺🇦 Українська](../uk/README.md)*

SplineCraft Flow is a plugin for quickly building constructions along a spline out of static meshes and
actors: roads, rails, pipes, fences, curbs, bridges, tracks, rows of columns, lamps, or trees. All
configuration is done on a single `ASplineCraftFlowActor` actor through the **Details** panel (category
**SplineCraft Flow**), and the geometry rebuilds automatically after every property change.

> **SplineCraft Flow 7.0.0**, for Unreal Engine 5.8. What's new in this release — [Release Notes](Release-Notes.md).

## Quick Start

1. **Install the plugin** — from Fab / Epic Games Launcher (*Install to Engine*) or by copying the
   `SplineCraftFlow` folder into `<Project>/Plugins/`.
2. **Enable it** — *Edit → Plugins → SplineCraft Flow → Enabled*, restart the editor.
3. **Place an actor** — `BP_SplineCraftFlowActor` from the plugin content, or the **Spline Craft Flow Actor**
   class from the *Place Actors* panel.
4. **Set the spline shape** — move points in the viewport, or apply the modifier stack in
   **Spline Geometry Edit Mode** (rectangle, ellipse, spiral, road, loops, etc).
5. **Add a mesh** — in the **Static Meshes** array create an entry and assign a **Static Mesh**. The
   construction builds immediately.

Details — [01. Quick Start](01-Quick-Start.md).

## Glossary

- **Mesh entry** (`FSPMeshConfiguration`) — one "layer" of identical meshes along the whole spline with its
  own spacing, scale, material, and randomization settings. There can be any number of entries.
- **Segment** — one mesh instance within an entry: in **Line** mode this is an instance, in **Curve** mode
  it's a separate bent `SplineMeshComponent`.
- **Line / Curve mode** — the placement method: straight instances vs. meshes bent along the curve.
- **Actor entry** (`FSPSplineActorConfig`) — actors of a given class, spaced out along the spline.
- **Distance range** (`FSPDistanceRange`) — a section of the spline in percent or meters; used for hidden
  ranges and materials.
- **Geometry modifier** — a step in the **Spline Geometry Edit Mode** stack that builds or changes spline
  points.
- **Preset** (`USplineCraftFlowPreset`) — a reusable generation style without the spline shape or **Loop**.

## Sections

| # | Section | What's inside |
|---|--------|--------------|
| – | [Release Notes](Release-Notes.md) | What each version adds, fixes, and what to watch for when upgrading |
| 01 | [Quick Start](01-Quick-Start.md) | Installation, enabling, first construction |
| 02 | [Spline and Placement Strategies](02-Spline-And-Placement-Strategies.md) | Manual editing, 8 strategies, redistribution, alignment, copying the spline |
| 03 | [Static Meshes](03-Static-Meshes.md) | Mesh entry parameters: spacing, offsets, scale, rotation, twist, collision |
| 04 | [Line and Curve Modes](04-Line-vs-Curve-Mode.md) | Global actor mode and entry mode; how the mesh is placed |
| 05 | [Distance Ranges and Hidden Ranges](05-Distance-Ranges.md) | `FSPDistanceRange`, **Hidden Ranges** |
| 06 | [Materials](06-Materials.md) | **Materials**, **Materials Distance Ranges**, **Slot Materials**, priorities |
| 07 | [Randomization and Random Seed](07-Randomization.md) | Scale, offset, rotation spread, Flip; reproducible seed |
| 08 | [Actors Along the Spline](08-Actors-Along-Spline.md) | Spacing, offset, rotation, scale, vertical alignment |
| 09 | [Transform, Mobility, and Blueprint API](09-Transform-Mobility-And-Blueprint.md) | Actor rotation and scale, **Scale Structure With Actor**, runtime rebuilding |
| 10 | [Merge to Static Mesh](10-Merge-To-Static-Mesh.md) | **Merge to Static Mesh**, **Replace Source Actor**, Nanite |
| 11 | [Performance](11-Performance.md) | What the construction costs, limits, tips |
| 12 | [FAQ](12-FAQ.md) | Short answers to common questions |
| 13 | [Generation Style Presets](13-Presets.md) | Creating, applying, and updating reusable Data Asset presets |

## Requirements

- Unreal Engine **5.8**

## Support

- Discord: https://discord.gg/3cyFmzgnNR
