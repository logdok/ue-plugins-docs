# Randomization and Random Seed

*🇬🇧 English | [🇺🇦 Українська](../uk/08-Randomization.md)*

Randomization settings add a controlled spread of scale, paddings, shift and rotation between
instances. Each parameter is a `From`..`To` range; a value is taken uniformly from it.

## Spread for sections and tubes (`FSCElementRandomSettings`)

| Parameter | What it varies |
|---|---|
| **Scale By Thickness (Y)** / **Scale By Height (Z)** | mesh scale. |
| **Left / Top / Right / Bottom Padding** | paddings (cm). |
| **Shift Horizontal (Y)** / **Shift Vertical (Z)** | offset (cm). |
| **Roll Angle (X)** | tilt around the element's axis (degrees). |
| **Rotate 0 or 180** | a random 0° or 180° turn. |

## Spread for posts and knobs (`FSCPointRandomSettings`, `FSCPointKnobRandomSettings`)

| Parameter | What it varies |
|---|---|
| **X / Y / Z Scale** | scale per axis. |
| **X / Y / Z Shift** | offset per axis (cm). |
| **Roll (X) / Pitch (Y) / Yaw (Z) Angle** | angles (degrees). |
| **Flip X** / **Flip Y** | a random 180° turn. |

> In **Curve** mode the **Pitch**, **Yaw** and **Flip** parameters have no effect.

## Ignore flags

At the mesh-configuration level: **Ignore Scaling / Padding / Translating / Rotation
Randomization** — turn off the corresponding spread group for this particular mesh, without
touching others.

## Random Seed

By default the spread is re-rolled on every rebuild — so two identically configured actors
look different, and a variation you liked cannot be pinned down. The **Randomization** section
on the actor solves this:

| Parameter | Purpose |
|---|---|
| **Use Random Seed** | turn on deterministic mode. |
| **Random Seed** | the seed: with one seed the structure is built identically every time. |
| **Randomize Seed** (button, editor only) | pick a new seed and rebuild. |

The seed covers **everything**: scale, paddings, shift, rotation, Flip and the random mesh
choice. By default **Use Random Seed** is off — the legacy behaviour.

> **Note.** Changing settings that change the **number** of random draws (adding a section,
> enabling another randomization) shifts the sequence, so the look at the same seed changes —
> that is a property of the approach itself.

## When it is recomputed

- `OnConstruction` — on every property change in the editor.
- `BeginPlay` — at the start of play.
- At runtime — via `UpdateAfterChangeAnyProperty()`, if **Mobility ≠ Static**
  (see [chapter 10](10-Runtime-And-Blueprint-API.md)).

## See also

- [Sections & Tubes](04-Sections-And-Tubes.md)
- [Posts](03-Posts.md)
