# Line and Curve Modes

*🇬🇧 English | [🇺🇦 Українська](../uk/04-Line-vs-Curve-Mode.md)*

The mode determines which engine components the generated meshes end up in, and whether they bend along
the curve.

## Global Actor Mode (`Mode`, `ESPUsingSpawnMode`)

| Value | Effect |
|---|---|
| **Use Concrete Mode in Mesh Configuration** | each mesh entry decides for itself via its own **Mode** field (default value). |
| **Line** | all entries — straight instances in an `InstancedStaticMeshComponent`. Best performance. |
| **Curve** | all entries — separate `SplineMeshComponent`s that bend along the curve. |
| **Visible Only Spline** | nothing is created — only the spline. Handy for debugging the shape; actors along the spline don't appear either. |

## Entry Mode (`Mode`, `ESPSpawnMode`)

Each entry in **Static Meshes** has its own **Mode** field: `Line` or `Curve` (default `Curve`). It only
takes effect when the actor's global mode is `Use Concrete Mode in Mesh Configuration`. This lets you
keep straight slabs in Line and a flexible pipe in Curve within the same construction.

## Line

- Each segment is a mesh instance at a spline point, rotated to match the spline's direction.
- Instances of one entry with the same materials are collected into a single component and added in one
  batch — hundreds of meshes cost only a few draw calls.
- The mesh **doesn't bend**: on curved sections these are straight pieces rotated along the curve.
- **Vertical Align**, full **Additional Rotation**, **Flip X / Flip Y** are available.
- The instance's pivot is the segment's center (segment start + half the mesh length + X offset).

## Curve

- Each segment is a separate `SplineMeshComponent` that deforms the mesh between the segment's start and
  end.
- The mesh follows the curve's bend — suitable for pipes, rails, handrails, roads on turns.
- **Additional Scale X** stretches the mesh along the spline; Y and Z scale the cross-section, also
  accounting for the spline point scale at the segment's start and end.
- **Additional Rotation** is Roll only; **Twist Amount** twists the mesh around the curve.
- The number of components grows fast: 200 segments = 200 components.
- For a smooth result the mesh needs enough detail along the X axis (several edge loops), otherwise it
  bends "faceted".

## Which to Choose

| Situation | Mode |
|---|---|
| Straight elements, fences, posts, sleepers, most scenes | **Line** |
| The mesh needs to bend along the curve (pipes, rails, road surface) | **Curve** |
| Mixed construction | **Use Concrete Mode in Mesh Configuration** + **Mode** on entries |
| Large final construction | either mode + [Merge to Static Mesh](10-Merge-To-Static-Mesh.md) + Nanite |

## See Also

- [Static Meshes](03-Static-Meshes.md)
- [Performance](11-Performance.md)
