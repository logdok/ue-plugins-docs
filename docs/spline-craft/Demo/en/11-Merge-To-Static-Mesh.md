# Merge to Static Mesh

*🇬🇧 English | [🇺🇦 Українська](../uk/11-Merge-To-Static-Mesh.md)*

> ⚠️ **This is the SplineCraft Demo (evaluation) edition.** Every feature works exactly like the full plugin; only *how much* you can build with it is capped, an on-screen watermark is shown, and it does nothing in Shipping builds. Full list: [Demo Limitations](00-Demo-Limitations.md).

**Merge to Static Mesh** is an editor-only action. It bakes all of the actor's generated
geometry — posts, sections, tubes, knobs and polygons — into a single `StaticMesh` asset and
places a `StaticMeshActor` with it in the level.

This is the **recommended path for large static structures** — long fences, railings, roads,
tracks, walls — once the parameters no longer need to change. Instead of hundreds of
components you get one mesh that the engine treats as ordinary static geometry.

## Workflow

1. **Fully configure the structure** — the spline shape, posts, sections, knobs, polygons.
   After the merge the parameters are no longer editable (see "Limitations").
2. **Set the Merge Settings** — these are the engine's standard `FMeshMergingSettings`:
   material merging and simplification, lightmap UV generation, and so on. The defaults are
   fine for most cases.
3. **Press `Merge to Static Mesh`.** A confirmation dialog appears. A new `StaticMesh` asset
   is created under `/Game/Meshes/`, and a `StaticMeshActor` is added to the level.
4. If needed, turn on **Replace Spline Actor** *before* the merge — then the SplineCraft Demo actor
   is removed and only the baked mesh is left in its place.
5. **Enable Nanite** on the resulting `StaticMesh` (in its settings). Nanite handles culling
   and level of detail at any distance itself — no manual LODs required. For a large fence
   this is the best result for performance.

## Parameters

- **Merge Settings** (`FMeshMergingSettings`) — material simplification, lightmaps, merge
  options.
- **Replace Spline Actor** — remove the spline actor after the merge and keep only the mesh.

Both parameters are only available on a level instance of the actor (not on the Blueprint
class).

## What ends up in the result

- Posts, sections, tubes and knobs — from both modes, Line and Curve.
- **Polygons** — the polygons' procedural mesh is baked into a temporary static mesh before
  the merge and is included in the result too.
- **Build Scenario preview** — if a scenario is assigned and its **Preview Progress** is
  scrubbed up so geometry is shown, that previewed geometry is baked in as well. This is how
  you get a single mesh for a structure you designed as a growing scenario: scrub the preview
  to `1`, then merge.
- The opening markers in the viewport are a pure editor highlight and do not make it into the
  merge.

If there is nothing to bake — no mesh configured and no scenario preview shown — the action
reports it with a message instead of doing nothing silently.

## Limitations

- **Irreversible.** The baked mesh is not linked to the actor. To change the fence you have
  to bring the SplineCraft Demo actor back (if Replace Spline Actor was on — from history or from
  scratch) and merge again.
- **One big mesh.** Occlusion becomes coarser than when each section is its own component: the
  engine hides the mesh entirely or not at all. For very long structures, split into several
  actors and merge each separately.
- The collision and lightmaps of the baked mesh depend on the Merge Settings and on the
  `StaticMesh`'s own settings — check them after the merge.

## See also

- [Performance and Baking](13-Performance-And-Baking.md)
- [Line vs Curve Mode](09-Line-vs-Curve-Mode.md)
