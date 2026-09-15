# Transform, Mobility, and Blueprint API

*🇬🇧 English | [🇺🇦 Українська](../uk/09-Transform-Mobility-And-Blueprint.md)*

## Actor Rotation, Translation, and Scale

All generated geometry lives in the actor's local space:

- **Translating and rotating** the actor moves the construction as a whole — in the editor and in the
  game. After a rebuild, meshes and actors stand in the same places relative to the actor.
- **Scaling** the actor scales the construction as a whole: the number and spacing of elements don't
  change, meshes get bigger or smaller (when **Scale Structure With Actor** is enabled, below).
- Generated components are attached to the `SplineComponent`, so if you move or rotate the spline
  component itself inside the actor, the content moves along with the spline.

> The engine can't precisely convey a non-uniform actor scale (e.g. 2 × 1 × 1) to rotated child actors —
> for actors along the spline, keep the scale uniform.

### Scale Structure With Actor

| Value | Behavior |
|---|---|
| **Enabled** (default) | distances along the spline are computed in the actor's local units. Actor scale scales the whole construction. |
| **Disabled** | pre-7.0 behavior: distances are computed with the spline's world scale, so at actor scale 2 there are twice as many meshes along the spline, and they overlap. |

For actors on levels saved in versions before 7.0, this parameter is automatically disabled on load, so
existing scenes look the same as before. If such an actor is scaled and you want the new behavior, enable
the checkbox manually. At scale 1, both variants give the same result.

## Mobility

The actor's **Mobility** sets the mobility of the root, the spline component, and all generated
components.

| Value | When to use |
|---|---|
| `Static` (default) | stationary constructions; components take part in lighting bakes. |
| `Stationary` | components that don't move in the game (behavior matches the engine's standard Mobility). |
| `Movable` | the actor moves or rotates in the game. |

Runtime rebuilding works with any value, but components created during the game don't have baked
lighting.

## Rebuilding

- **In the editor** — automatically after every property change, actor move, or spline edit.
- **At game start** — in `BeginPlay`.
- **At runtime** — call **Build Elements Along Spline** after changing properties.

Rebuilding only destroys components that the actor itself created. Components you added in a Blueprint
subclass or on an instance remain.

## Blueprint API

| Element | Type | Purpose |
|---|---|---|
| **Build Elements Along Spline** | function | fully rebuild the construction using the current settings. |
| **Copy Spline From Actor** | function | copy the spline from another actor's first spline component (points, tangents, types, roll, scale, **Loop**) and move the actor so the splines match in world space. Returns `false` if the actor has no spline or is the actor itself. |
| **Apply Preset** | function | apply a `USplineCraftFlowPreset` and rebuild the construction without changing the spline's shape or **Loop**. |
| **Preset**, **Mode**, **Static Meshes**, **Actors**, **Mobility**, **Loop**, **Scale Structure With Actor**, **Stable Curve Orientation** | properties | read and write; call **Build Elements Along Spline** after changing a property directly. |
| **Spline Component**, **Default Scene Root** | properties | actor components (read-only). |

The editor-only commands **Align End to Start**, **Redistribute to N Points**, preset creation/saving, and
**Merge to Static Mesh** are deliberately not properties of the runtime actor. For runtime geometry work
through the standard Blueprint `Spline Component` API, then call **Build Elements Along Spline**.

## Example: A Road Between Two Points at Runtime

1. Get the **Spline Component**, clear its points (*Clear Spline Points*), and add new ones
   (*Add Spline Point*).
2. Call **Build Elements Along Spline**.
3. If the road should later move with the actor, set **Mobility** = `Movable`.

## See Also

- [Spline and Placement Strategies](02-Spline-And-Placement-Strategies.md)
- [Generation Style Presets](13-Presets.md)
- [Performance](11-Performance.md)
