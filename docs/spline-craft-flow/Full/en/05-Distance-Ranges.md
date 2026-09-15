# Distance Ranges and Hidden Ranges

*🇬🇧 English | [🇺🇦 Українська](../uk/05-Distance-Ranges.md)*

Some settings apply not to the whole spline but to specific sections. A section is defined by the
**distance range** structure (`FSPDistanceRange`). It's used by mesh and actor **Hidden Ranges**,
**Materials Distance Ranges**, and **Slot Materials**.

## Distance Range (`FSPDistanceRange`)

| Parameter | Purpose |
|---|---|
| **Begin Of Distance** | start of the section, as a percentage of the spline's length (0–100). |
| **Distance (m)** | length of the section in meters from the start. Set to 0 to specify length as a percentage instead. |
| **Length (%)** | length of the section as a percentage of the spline's length. Takes effect only when **Distance (m)** = 0. |

- **Distance (m)** and **Length (%)** are mutually exclusive: while one is nonzero, the other is locked.
- If both are 0, the range is disabled.
- The section covers distances from `Begin` to `Begin + length`, inclusive.
- By default **Distance (m)** = 1, so a new range is 1 meter from the start of the spline.
- Meters are counted in the actor's local units: if the actor is scaled, the section scales along with
  the construction (when **Scale Structure With Actor** is enabled, see
  [section 09](09-Transform-Mobility-And-Blueprint.md)).

## When an Element Falls Inside a Range

A single distance along the spline is checked — the element's **reference point**:

| Element | Reference point |
|---|---|
| Mesh in **Line** mode | segment center (accounting for the X offset). |
| Mesh in **Curve** mode | segment start. |
| Actor along the spline | actor index × step (not counting the X offset from **Offset Transform**). |

An element falls inside a range if its reference point lies within it. A long mesh that only partially
overlaps a range doesn't belong to it.

## Hidden Ranges

- **Static Meshes → Hidden Ranges** — segments of an entry whose reference point lies in any of the
  ranges are not created.
- **Actors → Hidden Ranges** — the same for actors along the spline.

Hidden segments don't leave "holes" in the numbering: neighboring segments stay in their places, so
hiding is used for driveways, gates, fence gaps. For a clean driveway edge, place spline points along its
boundaries.

## Examples

| Task | Settings |
|---|---|
| Remove meshes over 5 meters in the middle of the spline | **Begin Of Distance** = 50, **Distance (m)** = 5 |
| Remove the last quarter of the spline | **Begin Of Distance** = 75, **Distance (m)** = 0, **Length (%)** = 25 |
| Different material on the first 10 meters | an entry in **Materials Distance Ranges** with range **Begin** = 0, **Distance (m)** = 10 |

## See Also

- [Materials](06-Materials.md)
- [Actors Along the Spline](08-Actors-Along-Spline.md)
