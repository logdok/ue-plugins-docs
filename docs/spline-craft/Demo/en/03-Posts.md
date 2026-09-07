# Posts

*🇬🇧 English | [🇺🇦 Українська](../uk/03-Posts.md)*

> ⚠️ **This is the SplineCraft Demo (evaluation) edition.** Every feature works exactly like the full plugin; only *how much* you can build with it is capped, an on-screen watermark is shown, and it does nothing in Shipping builds. Full list: [Demo Limitations](00-Demo-Limitations.md).

The **POSTS** section places a static mesh at each active spline point: a fence post, a
baluster, a column, a stanchion. One entry in the `Posts` array is one "layer" of posts along
the whole spline; there can be several entries (for example a post plus a separate base).

## Post (`FDemoSCPoint`)

| Parameter | Purpose |
|---|---|
| **Distance From Bottom** | height above the spline (cm). |
| **Visible All** | switch for the whole entry. |
| **Main Static Mesh Configuration** | the main mesh configuration (below). |
| **Detailed Visibility Configuration** | per-entry visibility rules — see [Visibility Rules](07-Visibility-Rules.md). |
| **Knobs** (`Point Knobs`) | array of knobs mounted on the post — see [Knobs](05-Knobs-And-Free-Knobs.md). |
| **Visible Knobs** | switch for all of this post's knobs. |

## Main Static Mesh Configuration (`FDemoSCPointMeshConfig`)

| Parameter | Purpose |
|---|---|
| **Static Mesh** | the post mesh. |
| **Visible** | show the mesh. |
| **Scale** | scale (`FVector`). |
| **Shift** | offset (`FVector`, cm): X along the spline, Y sideways, Z up. |
| **Rotation** | an added rotation. |
| **Displayed Object** (`EDemoShowKnobMode`) | `Mesh` — show a mesh; `Actor` — spawn an actor (visible in game only). See [Additional Actors](12-Additional-Actors.md). |
| **Actor** (`FDemoSCAdditionalActor`) | the actor class when `Actor` mode is chosen. |
| **Ignore … Randomization Settings** | turn off scale / translation / rotation randomization separately. |
| **Randomization Settings** | spread of scale, shift and angles — see [Randomization](08-Randomization.md). |

**Lateral shift (Shift.Y)** is applied as a parallel offset (a mitre): a shifted row of posts
stays strictly parallel to the spline even through corners, instead of being pulled inwards.

## Alternate Meshes

Swap the main mesh by point position: **First**, **Last**, **Odd**, **Even**. It applies when
the matching configuration has a valid mesh (or actor class) and `Visible` is on. Turned off
with **Ignore Alternate Meshes**.

## Special Meshes

A list of configurations, each with a set of **Visible At Index Ranges**: at points with those
indexes the main mesh is replaced by the special one. It takes priority over alternate and
random meshes. Turned off with **Ignore Special Meshes**.

## Random Meshes

Turn on **Use Random Meshes** and fill the **Random Meshes** list — at each point a mesh is
picked at random from the valid ones. For a reproducible result set a
[Random Seed](08-Randomization.md).

## Mesh selection order

Special → random → alternate → main. The first one that matches is used; its `Shift` is added
to the main configuration's `Shift`.

## Visibility

- **Visible All**, **Visible All Knobs** — at the `POSTS` section level.
- **All Posts Visibility Configuration** — visibility rules for all posts at once.
- The last point of an open spline is shown only if **Visible Last Point Elements** is on.

Full logic is in [Visibility Rules and Openings](07-Visibility-Rules.md).

## See also

- [Knobs and Free Knobs](05-Knobs-And-Free-Knobs.md)
- [Randomization and Random Seed](08-Randomization.md)
