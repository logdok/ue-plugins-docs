# FAQ

*🇬🇧 English | [🇺🇦 Українська](../uk/12-FAQ.md)*

## Meshes don't appear

Check: a **Static Mesh** is assigned in the entry; the actor's **Mode** isn't `Visible Only Spline`; the
section isn't hidden via **Hidden Ranges**; the step (mesh length + **Spacing**) is positive — otherwise
there will be a warning in the `LogSplineCraftFlow` log.

## The mesh doesn't bend along the curve

That's **Line** mode — the mesh sits as a straight piece. Switch the entry (or the whole actor) to
**Curve** mode. See [Line and Curve Modes](04-Line-vs-Curve-Mode.md).

## In Curve mode the mesh bends in a faceted way

The mesh doesn't have enough geometry along the X axis. Add edge loops along the mesh's length.

## I rotated or scaled the actor — will the construction change?

No: the construction rotates and scales as a whole, the number and position of elements relative to the
actor don't change. For actors from scenes saved before version 7.0, scaling behaves as before — see
**Scale Structure With Actor** in [section 09](09-Transform-Mobility-And-Blueprint.md).

## How do I reuse a shape or a style?

For the shape, open **Spline Geometry Edit Mode** and set up the modifier stack. For meshes, actors,
materials, and the generation mode, create a [SplineCraft Flow Preset](13-Presets.md). A preset doesn't
change the spline's points or **Loop**, so shape and style can be combined independently.

## A closed spline opens up on its own

Closedness is controlled by the **Loop** checkbox on the actor, not the spline component's flag — the
actor syncs it on every rebuild.

## How do I make a driveway or a gap in a fence?

Add a range to the mesh entry's **Hidden Ranges** (and, if needed, the actors'). For a clean driveway
edge, place spline points along its boundaries. See [section 05](05-Distance-Ranges.md).

## Slot Materials aren't working

The **Slot Materials** entry needs at least one range in **Distance Ranges**, and **Material Slot Name**
must match the mesh's slot name. See [Materials](06-Materials.md).

## Two identically configured entries look the same — but I need variety

Give the entries different **Random Seed** values. See [Randomization](07-Randomization.md).

## My components in a Blueprint subclass disappeared after a rebuild

In versions before 7.0, a rebuild removed all ISM, Spline Mesh, and Child Actor components on the actor.
Starting with 7.0, only components created by the actor itself are removed.

## How do I make a large road or fence perform well?

Build the construction, run [Merge to Static Mesh](10-Merge-To-Static-Mesh.md), and enable Nanite on the
resulting mesh. See [Performance](11-Performance.md).

## See Also

- [Quick Start](01-Quick-Start.md)
- [Release Notes](Release-Notes.md)
