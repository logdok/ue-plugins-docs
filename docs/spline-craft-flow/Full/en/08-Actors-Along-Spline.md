# Actors Along the Spline

*🇬🇧 English | [🇺🇦 Українська](../uk/08-Actors-Along-Spline.md)*

The **Actors** array on the actor places full-fledged actors of any class along the spline: lamps with
logic, signs, triggers, Blueprint actors with effects. Each entry (`FSPSplineActorConfig`) is a row of
actors of one class with a uniform step.

## Parameters

| Parameter | Purpose |
|---|---|
| **Design Notes** | a free-form note. |
| **Actor Class** | the actor class. An entry with no class is ignored. |
| **Gap between Pivot Points** | the step between actors along the spline, cm. Actors are placed at distances 0, step, 2 × step … up to and including the end of the spline. 0 — a single actor at the start of the spline. |
| **Offset Transform** | offset, rotation, and scale of the actors. **Translation X** — offset along the spline (not past the spline's ends), **Y** — to the right, **Z** — upward. Rotation is added to the spline's rotation in the actor's local axes. |
| **Scale Multiplier** | a multiplier on top of the scale from **Offset Transform**. |
| **Twist Amount** | twist: number of full roll turns around the spline over its whole length. |
| **Vertical Align** | keep actors vertical: the spline's tilt and roll and **Offset Transform**'s rotation are ignored, only the direction remains. |
| **Hidden Ranges** | sections where actors aren't created — see [section 05](05-Distance-Ranges.md). |

## How It Works

- Each actor is created via a `ChildActorComponent` attached to the spline. Components have stable names
  `ChildActor_<entry number>_<actor number>`.
- Actors exist both in the editor and in the game; they're visible in the viewport right away.
- On every rebuild the actors are recreated. Don't edit them by hand in the scene — the changes will be
  lost. Configure the class's Blueprint or the entry's parameters instead.
- Y and Z offsets are computed relative to the spline's horizontal direction and the actor's Z axis.
- **Visible Only Spline** mode disables actors too.

## Merging

**Merge to Static Mesh** adds the child actors' static meshes (`StaticMeshComponent`s) to the result.
Logic, lights, and other actor components don't take part in the merge — see
[section 10](10-Merge-To-Static-Mesh.md).

## See Also

- [Distance Ranges and Hidden Ranges](05-Distance-Ranges.md)
- [Transform, Mobility, and Blueprint API](09-Transform-Mobility-And-Blueprint.md)
