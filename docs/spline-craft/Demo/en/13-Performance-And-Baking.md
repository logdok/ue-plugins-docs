# Performance and Baking

*🇬🇧 English | [🇺🇦 Українська](../uk/13-Performance-And-Baking.md)*

> ⚠️ **This is the SplineCraft Demo (evaluation) edition.** Every feature works exactly like the full plugin; only *how much* you can build with it is capped, an on-screen watermark is shown, and it does nothing in Shipping builds. Full list: [Demo Limitations](00-Demo-Limitations.md).

## What a structure costs

The **Stats** section in the Details panel shows the result of the last build: how many
instanced components, how many instances in total, how many Spline Mesh components and how
many procedural meshes (polygons). Look there when a structure starts to feel heavy.

## Line vs Curve

- **Line** — all meshes go into an `InstancedStaticMeshComponent`, one component per unique
  mesh. Cheap, and the whole structure stays editable. Fine for most scenes.
- **Curve** — each element of each span becomes its own `SplineMeshComponent`. The component
  count climbs fast: a second section on a 50-point spline is another ~50 components. Turn
  Curve on only where the mesh really must bend along the curve, and watch Stats.

## Large static structures: build and bake

For long fences, railings, roads and tracks — once the parameters are final — the
**recommended path is [Merge to Static Mesh](11-Merge-To-Static-Mesh.md)**:

1. Fully configure the structure.
2. Run the merge — all the geometry becomes one `StaticMesh` asset.
3. Enable **Nanite** on it.

After that the engine treats the structure as an ordinary static mesh with Nanite:
per-cluster culling and level of detail at any distance, no manual LODs and no instancing
cost. This is almost always a better result than trying to keep thousands of Spline Mesh
components alive.

The downside is that the structure "freezes": to change it you bring the SplineCraft Demo actor
back and merge again.

## A note on the number of unique meshes

In Line mode components are keyed by "mesh + materials". The fewer **different** meshes and
materials in a structure, the fewer components and draw calls. Random meshes from a large
list multiply the number of ISM components.

## Lighting and baking

- **Mobility = Static** — required for components to take part in static lighting bakes.
- The **Build Scenario** step child actors are forced `Movable` — they will not receive
  baked static light. For a "frozen" result use a normal actor without a scenario, or Merge
  to Static Mesh.
- For the final build of large structures, Merge to Static Mesh plus baked lighting gives the
  most predictable result.

## See also

- [Line vs Curve Mode](09-Line-vs-Curve-Mode.md)
- [Merge to Static Mesh](11-Merge-To-Static-Mesh.md)
