# Quick Start

*🇬🇧 English | [🇺🇦 Українська](../uk/01-Quick-Start.md)*

This section walks you from installing the plugin to your first generated construction in the scene.

## Installation

**From Fab / Epic Games Launcher.** Find SplineCraft Flow in your library, click *Install to Engine* and
choose engine version 5.8. The plugin installs into the engine folder.

**Manually.** Copy the `SplineCraftFlow` folder into `<Project>/Plugins/` (create `Plugins` if it doesn't
exist). This way the plugin lives inside the project and travels with it in version control.

## Enabling the Plugin

1. **Edit → Plugins**.
2. Search for `SplineCraft Flow`.
3. Check **Enabled**.
4. Restart the editor when prompted.

To see the plugin content in the Content Browser, enable *Settings → Show Plugin Content*.

## First Construction in 5 Steps

1. **Place an actor.** Drag `BP_SplineCraftFlowActor` onto the scene
   (*Plugins → SplineCraft Flow Content → Blueprint*) or the **Spline Craft Flow Actor** class from the
   *Place Actors* panel. The actor has a `DefaultSceneRoot` root and a `SplineComponent` with two points.
2. **Set the spline shape.** Select the actor and move the spline points in the viewport (Alt-drag adds a
   point). For a ready-made shape, open the **Spline Geometry** editor mode, add, for example, the
   **Ellipse** modifier, review the preview, and click **Apply**. Details —
   [Spline and Placement Strategies](02-Spline-And-Placement-Strategies.md).
3. **Add a mesh.** In the **Static Meshes** array click **+**, expand the entry, and assign a
   **Static Mesh**. The meshes lay out along the entire spline.
4. **Choose a mode.** The actor's **Mode** field: `Line` — straight instances (fastest), `Curve` — meshes
   bend along the curve, `Use Concrete Mode in Mesh Configuration` — each entry decides for itself via its
   own **Mode** field. Details — [Line and Curve Modes](04-Line-vs-Curve-Mode.md).
5. **Tune the look.** **Spacing**, **Additional Offset**, **Additional Scale**, materials,
   randomization, actors along the spline — see sections [03](03-Static-Meshes.md)–[08](08-Actors-Along-Spline.md).

## What to Know Right Away

- **The geometry rebuilds itself** after every property change in the editor (`OnConstruction`) and at
  game start (`BeginPlay`). There's nothing to "apply."
- **Spline Geometry works as a modifier stack.** The order of steps affects the result; the preview
  doesn't change the actual spline until you click **Apply**.
- **A closed spline** is enabled with the **Loop** checkbox on the actor (not on the spline component —
  the actor overwrites that flag on every rebuild).
- **Actor rotation and scale** don't change the layout relative to the actor — the construction rotates
  and scales as a whole. Details — [section 09](09-Transform-Mobility-And-Blueprint.md).
- For large final constructions — [Merge to Static Mesh](10-Merge-To-Static-Mesh.md).
- To reuse a tuned style — [SplineCraft Flow Preset](13-Presets.md).

## What's Next

- [Spline and Placement Strategies](02-Spline-And-Placement-Strategies.md)
- [Static Meshes](03-Static-Meshes.md)
- [Line and Curve Modes](04-Line-vs-Curve-Mode.md)
