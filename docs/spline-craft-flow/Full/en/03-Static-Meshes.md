# Static Meshes

*🇬🇧 English | [🇺🇦 Українська](../uk/03-Static-Meshes.md)*

The **Static Meshes** array on the actor is a list of mesh "layers". Each entry (`FSPMeshConfiguration`)
lays out one static mesh along the whole spline: the road surface, the left curb, the right curb, a rail,
a pipe. Entries are independent: each has its own spacing, offsets, materials, randomization, and mode.

## How a Mesh Fills the Spline

- **Mesh length** — the size of its bounding box along the X axis. The mesh should be modeled along the
  X axis, with the pivot at its center.
- **Step** along the spline = mesh length + **Spacing**. In **Curve** mode the length is multiplied by
  **Additional Scale X**.
- Number of segments = spline length / step (rounded down) + 1. The last segment may go past the end of
  the spline — in that case it's clamped to the end.
- Distance along the spline is measured in the actor's local units (see
  [section 09](09-Transform-Mobility-And-Blueprint.md)).

## Mesh Entry Parameters

| Parameter | Purpose |
|---|---|
| **Design Notes** | a free-form note for the designer. |
| **Static Mesh** | the mesh laid out along the spline. An entry with no mesh is ignored. |
| **Mode** | `Line` or `Curve` for this entry. Only takes effect when the actor's **Mode** is `Use Concrete Mode in Mesh Configuration`. See [section 04](04-Line-vs-Curve-Mode.md). |
| **Spacing** | gap between neighboring meshes, cm. A negative value means overlap. |
| **Vertical Align** | keep the mesh vertical (ignoring the spline's tilt and roll). **Line** mode only. |
| **Additional Offset** | offset, cm: X — along the spline, Y — to the right, Z — upward. |
| **Additional Scale** | additional scale. In **Curve** mode X also stretches the mesh along the spline (and the step). |
| **Additional Rotation** | additional rotation. **Line** — all axes; **Curve** — Roll only (roll around the spline). |
| **Twist Amount** | twist: number of full turns around the spline over its whole length. **Curve** mode only. |
| **Receive Decals** | whether the meshes receive decals. |
| **Materials** / **Apply Every Index** | material overrides — see [Materials](06-Materials.md). |
| **Materials Distance Ranges** | materials on spline sections — see [Materials](06-Materials.md). |
| **Hidden Ranges** | sections where meshes aren't created — see [section 05](05-Distance-Ranges.md). |
| **Slot Materials** | material for a specific slot on given sections — see [Materials](06-Materials.md). |
| **Collision Enabled** | mesh collision mode (below). |
| **Randomization Settings** / **Random Seed** | scale, offset, and rotation spread — see [section 07](07-Randomization.md). |

## Offsets

- **Line.** The X offset moves the instance along the spline (by distance), Y and Z — from the point on
  the spline along its right and up directions. The Z offset is additionally multiplied by the spline
  point's scale (Z).
- **Curve.** The X offset shifts the whole segment along the curve (the ends are recomputed on the curve,
  not extrapolated), Y and Z — in the plane perpendicular to the curve.

## Collision

| Mode | Line (instances) | Curve (spline meshes) |
|---|---|---|
| `No Collision` | collision disabled, all channels — Ignore. | all channels — Ignore. |
| `Query Only` / `Physics Only` / `Query And Physics` | the collision profile comes from the mesh, the type from the parameter. | all channels — Block, type from the parameter. |

Static mesh collision depends on the simple collision configured on the mesh asset itself.

## Build Order

1. Mesh entries are processed top to bottom, segments — from the start of the spline.
2. In **Line** mode, instances with the same (entry, materials) are collected into a single
   `InstancedStaticMeshComponent`.
3. Generated components are attached to the `SplineComponent` and recreated on every rebuild. Components
   you added yourself (in a Blueprint subclass or on an actor instance) are left untouched.

## See Also

- [Line and Curve Modes](04-Line-vs-Curve-Mode.md)
- [Materials](06-Materials.md)
- [Randomization and Random Seed](07-Randomization.md)
