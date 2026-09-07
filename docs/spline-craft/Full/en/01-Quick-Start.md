# Quick Start

*🇬🇧 English | [🇺🇦 Українська](../uk/01-Quick-Start.md)*

This chapter takes you from installing the plugin to a first generated structure in the level.

## Installation

**From Fab / Epic Games Launcher.** Find SplineCraft in your library, click *Install to
Engine* and pick engine version 5.8. The plugin installs into the engine folder.

**Manual.** Copy the `SplineCraft` folder into `<Project>/Plugins/` (create `Plugins` if it
does not exist). This way the plugin lives inside the project and travels with it in source
control.

## Enabling the plugin

1. **Edit → Plugins**.
2. Search for `SplineCraft`.
3. Tick **Enabled**.
4. Restart the editor when prompted.

## A first structure in 5 steps

1. **Place the actor.** In the *Place Actors* browser find **SplineCraft Actor** and drag it
   into the level. The actor's root is a spline component with a few default points.
2. **Shape the spline.** Select the actor and move the spline points in the viewport, or open
   **SplineCraft Configuration → Utils → Alignment Spline Points**, pick a **Tool** (for
   example `Rectangle` or `Ellipse`), set **Num Spline Points** and the tool options. The
   shape rebuilds immediately. See
   [Spline Points and Alignment](02-Spline-Points-And-Alignment.md).
3. **POSTS.** Expand the **POSTS** section, add one entry to the `Posts` array, expand
   **Main Static Mesh Configuration** and assign a **Static Mesh** (e.g. `SM_Post_Sqr` from
   the plugin content). A post appears at each spline point.
4. **SECTIONS.** Expand **SECTIONS**, add an entry to the `Sections` array, and in
   **Main Static Mesh Configuration** assign a panel **Static Mesh** (e.g. `SM_Section`). The
   mesh stretches across each gap between points.
5. **KNOBS (optional).** Add an entry to the **KNOBS** section and assign a decorative mesh
   (e.g. `SM_Knob_Sphere`) — it becomes a finial on every post.

## Things to know right away

- **The geometry rebuilds itself** after every property change in the editor
  (`OnConstruction`). There is nothing to "apply".
- **In game** the structure only rebuilds if **Mobility ≠ Static**; after changing properties
  at runtime call `UpdateAfterChangeAnyProperty()` — see
  [Runtime, collision, events, presets, scenarios](10-Runtime-And-Blueprint-API.md).
- **Presets.** Instead of configuring by hand you can start from a ready style: assign a
  **SplineCraft Preset** asset to the **Preset** field and press **Apply Preset**. Example:
  `DA_Preset_IronFence` in `Content/Samples`.

## Next

- [Spline Points and Alignment](02-Spline-Points-And-Alignment.md)
- [Posts](03-Posts.md)
- [Sections & Tubes](04-Sections-And-Tubes.md)
