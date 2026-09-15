# Merge to Static Mesh

*🇬🇧 English | [🇺🇦 Українська](../uk/10-Merge-To-Static-Mesh.md)*

The **Merge to Static Mesh** tool bakes the generated geometry of each selected SplineCraft Flow actor
into a separate `StaticMesh` asset and places a matching `StaticMeshActor` in the scene. It's available
only in the editor, under **SplineCraft Flow Tools** in the **Details** panel, and doesn't add
editor-only properties to the runtime actor.

This is the **recommended final step for large constructions** — long roads, fences, tracks, and walls —
once their parameters no longer need to change. Instead of hundreds of components you get ordinary
static geometry that's easier to inspect, move, and optimize with the engine's own tools.

## Workflow

1. **Finish configuring the construction**: the spline's shape, meshes, materials, and actors along it.
2. Select one or more SplineCraft Flow actors in the scene. The batch command creates **one static mesh
   per actor**, not a single shared mesh for the whole selection.
3. In **SplineCraft Flow Tools → Merge to Static Mesh**, set the **Destination Folder**. By default this
   is `/Game/Meshes`; specify a content folder as a full package path starting with `/Game`.
4. If needed, expand **Advanced Merge Settings**. These are the engine's standard `FMeshMergingSettings`:
   material merging and simplification, lightmap UV unwrapping, **Pivot Point Type**, and so on. The
   defaults are enough for most cases.
5. Check the status line: it shows the number of eligible components, the destination folder, and the
   expected names. If the path is invalid or there's nothing to merge, the button is disabled and the
   reason is shown next to it.
6. Click **Create Static Mesh**, or, for multiple actors, **Create N Static Meshes**.
7. After creation, open the resulting assets and, if needed, enable **Nanite**, check collision and
   lightmaps.

Without **Replace Source Actor** the command needs no extra confirmation: the source actors stay in the
scene. During a large batch operation the editor shows progress; one actor's failure doesn't stop
processing the rest.

## Replacing Source Actors

Enable **Replace Source Actor** if, after a mesh is successfully created, the source SplineCraft Flow
actor should be removed from the scene. Before running, a confirmation appears with the asset count and
destination folder, and the panel separately warns about Undo behaving differently:

- creating the new `StaticMeshActor` and removing the source actor are undone by **Undo**;
- assets created in the Content Browser are **not deleted** by Undo — remove unwanted results manually;
- the whole batch creation and actor replacement is a single Undo step.

This is deliberately a **temporary safety toggle**. It isn't an actor property, isn't stored in the level
or a Blueprint, and isn't remembered as a user setting. The value resets after the command runs; it has
to be enabled again for the next replacement.

## Batch Operation Result

- For each eligible source actor, a separate asset `<Destination Folder>/SM_<actor name>` is created,
  along with a separate `StaticMeshActor` in the same level.
- If the name is already taken, the engine appends a number (`SM_<actor name>_1`, etc). Existing assets
  are never overwritten.
- Successfully created `StaticMeshActor`s become the new selection, and the Content Browser navigates to
  the created assets.
- An actor with no eligible static meshes is skipped; the batch continues, the summary shows the count of
  successful and skipped actors, and details go to the Output Log (`LogSplineCraftFlow`).
- If no mesh could be created at all, the actor transaction is rolled back and the panel shows an error.

## What Goes Into the Merge

- All valid components with a static mesh on the actor itself: **Line** mode instances, **Curve** mode
  spline meshes, and components you added yourself in a Blueprint subclass.
- Static mesh components of child actors placed along the spline (**Actors**).

Child actor logic, lights, Niagara, audio, and other non-static components are not merged.

## Settings and Persistence

| Parameter | Default | Persisted | Purpose |
|---|---|---|---|
| **Destination Folder** | `/Game/Meshes` | yes, separately per user and project | the Content Browser folder for new assets; a full package path without the asset name. |
| **Advanced Merge Settings** | engine defaults | yes, separately per user and project | `FMeshMergingSettings` applied to each actor in the batch. |
| **Replace Source Actor** | off | no | remove the corresponding source actor from the scene after a successful merge. |

The first two parameters survive an editor restart, but aren't serialized into `ASplineCraftFlowActor`
and don't change the level or a Blueprint. These are personal editor settings for the current project.
The riskier **Replace Source Actor** choice is deliberately never persisted.

## Limitations

- **The merged mesh isn't linked to the source actor.** To change the construction, edit the source actor
  (or bring it back via Undo after replacement) and create a new mesh.
- **One large mesh is baked as a whole.** Very long constructions are better split across several actors
  and baked in a batch: the result will still be separate for each one.
- Merge settings can significantly change materials, UVs, pivot, and collision. Check an important final
  result in the Static Mesh Editor.

## See Also

- [Performance](11-Performance.md)
- [Line and Curve Modes](04-Line-vs-Curve-Mode.md)
