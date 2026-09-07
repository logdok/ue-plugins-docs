# Knobs and Free Knobs

*🇬🇧 English | [🇺🇦 Українська](../uk/05-Knobs-And-Free-Knobs.md)*

> ⚠️ **This is the SplineCraft Demo (evaluation) edition.** Every feature works exactly like the full plugin; only *how much* you can build with it is capped, an on-screen watermark is shown, and it does nothing in Shipping builds. Full list: [Demo Limitations](00-Demo-Limitations.md).

**Knobs** are decorative meshes or actors: post finials, spheres, spires, lamps, signs.
SplineCraft Demo has three kinds.

## 1. The KNOBS section

The **KNOBS** section on the actor works exactly like [POSTS](03-Posts.md): a `Knobs` array
where each entry is an `FDemoSCPoint` that places a mesh at every active spline point. The
difference is only semantic — it is convenient to keep this section separate for a purely
decorative layer.

- **Visible All**, **Visible All Knobs**
- **All Knobs Visibility Configuration** — visibility rules for all entries of the section
- each `FDemoSCPoint` has its own **Point Knobs** array — knobs mounted on it

### Knob on a point (`FDemoSCPointKnob`)

| Parameter | Purpose |
|---|---|
| **Distance From Owner** | offset up along the owner's axis (cm). |
| **Main Static Mesh Configuration** (`FDemoSCPointKnobMeshConfig`) | **Scale**, **Shift**, **Rotation** (all `FVector` / `FRotator`), **Static Mesh**, **Displayed Object**, **Actor**. |
| **Alternate / Special / Random Meshes** | swap by position, by index ranges, at random. |
| **Detailed Visibility Configuration** | visibility rules — see [chapter 7](07-Visibility-Rules.md). |
| **Ignore … Randomization** / **Randomization Settings** | spread of scale and angles. |

## 2. Knobs on sections and tubes

These are set directly on the element (`FDemoSCElementKnob`) and attach to the span at the
weighted position **Weight Horizontal Position** (0..1). Described in
[Sections & Tubes](04-Sections-And-Tubes.md).

## 3. Free Knobs (the FREE KNOBS section)

`FDemoSCFreePointKnobs` / `FDemoSCFreePoint` — knobs **not bound to spline points**. They are placed
in the actor's local space: the position is set by **Distance From Bottom** (a Z rise) plus
the **Shift** from the mesh configuration. Good for one-off objects — a sign, a lamp, a single
detail next to the structure.

| Parameter | Purpose |
|---|---|
| **Visible** | show the knob. |
| **Distance From Bottom** | Z rise in the actor's local space (cm). |
| **Main Static Mesh Configuration** (`FDemoSCPointMeshConfig`) | Scale, Shift, Rotation, Static Mesh, Displayed Object, Actor. |
| **Use Random Meshes** + **Random Meshes** | random mesh selection. |

`FDemoSCFreePointKnobs` has a shared **Visible All** switch.

## Mesh or actor (`EDemoShowKnobMode`)

- **Mesh** — show a static mesh. Visible both in the editor and in game.
- **Actor** — spawn an actor of the given class. Visible **in game only**. The `Shift` and
  `Rotation` values come from the knob's settings. See [Additional Actors](12-Additional-Actors.md).

## See also

- [Posts](03-Posts.md)
- [Sections & Tubes](04-Sections-And-Tubes.md)
- [Randomization and Random Seed](08-Randomization.md)
