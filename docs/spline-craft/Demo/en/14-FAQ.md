# FAQ

*🇬🇧 English | [🇺🇦 Українська](../uk/14-FAQ.md)*

> ⚠️ **This is the SplineCraft Demo (evaluation) edition.** Every feature works exactly like the full plugin; only *how much* you can build with it is capped, an on-screen watermark is shown, and it does nothing in Shipping builds. Full list: [Demo Limitations](00-Demo-Limitations.md).

## The plugin used to be called "SplineCraft Demo PRO" — is it the same product?

Yes. The plugin was renamed to **SplineCraft Demo**. Compatibility is preserved: the module name
(`SplineCraftDemo`), the content mount point (`/SplineCraftDemo/...`), the class paths
(`/Script/SplineCraftDemo.*`) and the property layout are unchanged. Existing levels, Blueprint
subclasses of `ADemoSplineCraftActor` and asset references keep working. See
[Release Notes](Release-Notes.md).

## Why doesn't the structure update in game?

At runtime the geometry only rebuilds if **Mobility ≠ Static**. Set `Movable` or `Stationary`
and, after changing properties, call `UpdateAfterChangeAnyProperty()`.

## My spline snaps back to its previous shape the moment I drag a point

An alignment tool is selected in **Utils → Alignment Spline Points**. A tool runs when its
settings change. Switch **Tool** to `Manual` for fully manual work, or use the **Apply Point
Layout** button only when you actually want a fresh layout.

## The mesh doesn't bend along the curve

That is **Line** mode — the mesh fills the span with a straight segment. To bend along the
curve, put the element (or the whole actor) into **Curve** mode. See
[Line vs Curve Mode](09-Line-vs-Curve-Mode.md).

## How do I make a large fence or road performant?

Build the structure fully, then run **[Merge to Static Mesh](11-Merge-To-Static-Mesh.md)** and
enable **Nanite** on the resulting mesh. You get one static mesh that the engine culls and
LODs itself, with no manual LODs and no instancing cost. Do not try to keep thousands of
Spline Mesh components alive — the **Stats** section in Details will tell you when there are
too many.

## How do I freeze the result for release?

**Merge to Static Mesh** bakes all the geometry into one `StaticMesh` asset. Turn on
**Replace Spline Actor** so only the baked mesh is left in the level. After that the
structure's parameters are no longer editable — keep the source SplineCraft Demo actor if you plan
to change the fence later.

## The hit events (`OnSplineCraft…HitEvent`) don't fire

Turn on **Override Collision**, set **Generate Hit Events**, and choose a collision profile
with physics blocking (for example `BlockAll`). Without that the engine does not send hit
events. See [chapter 10](10-Runtime-And-Blueprint-API.md).

## Two identically configured actors look different

Randomization is re-rolled on every rebuild. Turn on **Use Random Seed** and set a
**Random Seed** — with one seed the structure is built identically. The **Randomize Seed**
button picks a new variation.

## How do I make a gate or a passage in a fence?

Use the **OPENINGS** section: set the opening's **Start** and **Width** in centimetres along
the spline. An opening hides whole elements, so for a clean gate edge place spline points at
its borders. See [Visibility Rules and Openings](07-Visibility-Rules.md).

## How do I move a style between levels or projects?

Use **Presets**: press **Save To Preset** to store the current settings into a
`SplineCraftDemo Preset` asset, and **Apply Preset** on another actor to load the style. A preset
does not touch the spline shape or the openings. See [chapter 10](10-Runtime-And-Blueprint-API.md).

## How do I make a building that rises floor by floor?

That is **Build Scenario**: an asset with a sequence of presets, each with its own Z offset.
A director actor spawns the steps one by one along its spline. `Reverse` takes the structure
back down. See [chapter 10](10-Runtime-And-Blueprint-API.md).

## See also

- [Quick Start](01-Quick-Start.md)
- [Release Notes](Release-Notes.md)
