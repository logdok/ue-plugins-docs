# Line vs Curve Mode

*🇬🇧 English | [🇺🇦 Українська](../uk/09-Line-vs-Curve-Mode.md)*

> ⚠️ **This is the SplineCraft Demo (evaluation) edition.** Every feature works exactly like the full plugin; only *how much* you can build with it is capped, an on-screen watermark is shown, and it does nothing in Shipping builds. Full list: [Demo Limitations](00-Demo-Limitations.md).

The mode decides which engine components the generated meshes go into.

## The actor's global mode (`Mode`, `EDemoSplineCraftUseMode`)

| Value | Effect |
|---|---|
| **Concrete** | each element decides for itself, via its own `Mode`. |
| **Line** | all meshes go into an `InstancedStaticMeshComponent`. Best performance. |
| **Curve** | section and tube meshes go into a `SplineMeshComponent` (they bend along the curve); posts and knobs still go into ISM. |

## The element mode (`EDemoSplineCraftMode`)

Every section or tube `Main Static Mesh Configuration` has its own **Mode**: `Line` or
`Curve`. It applies **only** when the actor's global **Mode** is `Concrete`. This lets one
structure keep straight panels in Line and a flexible railing in Curve.

## Line

- Meshes are collected into an `InstancedStaticMeshComponent`, one component per **unique
  "mesh + materials" pair**.
- Cheap: hundreds of posts are a handful of draw calls.
- The gap between points is filled by **stretching along a straight line** (a chord) between
  adjacent points. On a curved spline the span stays a straight segment.
- The whole structure stays editable.

## Curve

- Each element of each span becomes its own `SplineMeshComponent` that **bends along the
  curve**.
- The component count climbs fast: a second section on a 50-point spline is another ~50
  components. Watch the **Stats** section in Details.
- Needed where the mesh really must follow the bend: curved railings, tubes, rings.

## What to choose

| Situation | Mode |
|---|---|
| Straight fences, walls, panels, most scenes | **Line** |
| The mesh must bend along the curve | **Curve** |
| A mixed structure | **Concrete** + `Mode` on the elements |
| A large final structure | any mode + [Merge to Static Mesh](11-Merge-To-Static-Mesh.md) + Nanite |

## See also

- [Performance and Baking](13-Performance-And-Baking.md)
- [Merge to Static Mesh](11-Merge-To-Static-Mesh.md)
