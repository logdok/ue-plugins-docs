# SplineCraft User Guide

*🇬🇧 English | [🇺🇦 Українська](../uk/README.md)*

SplineCraft is a tool for quickly generating structures (compositions of `StaticMesh` and
`Actor` objects) along a spline: fences, railings, balustrades, colonnades, arcades,
barricades, bridges, roads and more. Everything is configured on a single `ASplineCraftActor`
through the **Details** panel (category **SplineCraft Configuration**), and the geometry
rebuilds automatically after every property change.

> For **Unreal Engine 5.8**. What's new in this release — [Release Notes](Release-Notes.md).

## Quick Start

1. **Install the plugin** — from Fab / Epic Games Launcher (*Install to Engine*), or by
   copying the `SplineCraft` folder into `<Project>/Plugins/`.
2. **Enable it** — *Edit → Plugins → SplineCraft → Enabled*, restart the editor.
3. **Drop a SplineCraft Actor** into the level (from the *Place Actors* browser).
4. **Shape the spline** — move the points in the viewport, or open
   **SplineCraft Configuration → Utils → Alignment Spline Points**, pick a **Tool**
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

## Plugin rename

The plugin was previously called **SplineCraft PRO**. It is now simply **SplineCraft**.
The rename **does not affect backward compatibility**: the module name (`SplineCraft`), the
content mount point (`/SplineCraft/...`), the class paths (`/Script/SplineCraft.*`) and the
layout of properties and structs (`FSC*`) are unchanged. Existing levels, Blueprint subclasses
of `ASplineCraftActor` and asset references keep working as-is. Only the plugin folder name and
the `FriendlyName` changed. The `BP_SplineCraft_PRO` asset was deliberately **not** renamed, so
references in projects do not break.

## Contents

| # | Chapter | What's inside |
|---|---------|---------------|
| – | [Release Notes](Release-Notes.md) | What each release adds, fixes and what to watch for |
| 01 | [Quick Start](01-Quick-Start.md) | Install, enable, a first structure in 5 steps |
| 02 | [Spline Points and Alignment](02-Spline-Points-And-Alignment.md) | Manual editing, 13 alignment tools, non-sticky tools |
| 03 | [Posts](03-Posts.md) | A mesh at each point; alternate / special / random meshes |
| 04 | [Sections & Tubes](04-Sections-And-Tubes.md) | `FSCElement`, filling the gap, paddings, knobs on the element |
| 05 | [Knobs and Free Knobs](05-Knobs-And-Free-Knobs.md) | Three kinds of knob; `Mesh` vs `Actor` mode |
| 06 | [Polygons](06-Polygons.md) | Filled shapes, `Follow Curve`, concave outlines, collision |
| 07 | [Visibility Rules and Openings](07-Visibility-Rules.md) | `FSCVisibleConfig` by index; `OPENINGS` by distance |
| 08 | [Randomization and Random Seed](08-Randomization.md) | Spread of scale / shift / rotation; a reproducible seed |
| 09 | [Line vs Curve Mode](09-Line-vs-Curve-Mode.md) | ISM vs Spline Mesh; global and per-element mode |
| 10 | [Runtime, collision, events, presets, scenarios](10-Runtime-And-Blueprint-API.md) | `UpdateAfterChangeAnyProperty`, Stats, `Collision`, hit events, presets, Build Scenario |
| 11 | [Merge to Static Mesh](11-Merge-To-Static-Mesh.md) | `Merge to Static Mesh`, `Replace Spline Actor`, Nanite |
| 12 | [Additional Actors](12-Additional-Actors.md) | `FSCAdditionalActor`, spawned in `BeginPlay` |
| 13 | [Performance and Baking](13-Performance-And-Baking.md) | What a structure costs; build and bake |
| 14 | [FAQ](14-FAQ.md) | Short answers to common questions |

## Requirements

- Unreal Engine **5.8**

## Support

- Discord: https://discord.gg/BQ69zYGy
