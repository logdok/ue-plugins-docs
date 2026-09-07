# Release Notes

*🇬🇧 English | [🇺🇦 Українська](../uk/Release-Notes.md)*

> ⚠️ **This is the SplineCraft Demo (evaluation) edition.** Every feature works exactly like the full plugin; only *how much* you can build with it is capped, an on-screen watermark is shown, and it does nothing in Shipping builds. Full list: [Demo Limitations](00-Demo-Limitations.md).

A short note on each release: what's new, what's fixed, and what to watch for when upgrading.
Newest release at the top.

---

## 7.0.0 — Unreal Engine 5.8

### Plugin rename

- The plugin was renamed from **"SplineCraft Demo PRO"** to **"SplineCraft Demo"**.
- **Backward compatibility is preserved.** Unchanged: the module name (`SplineCraftDemo`), the
  API macro (`SPLINECRAFTDEMO_API`), the content mount point (`/SplineCraftDemo/...`), the class paths
  (`/Script/SplineCraftDemo.*`), and the layout of properties and structs (`FSC*`). Existing
  levels, Blueprint subclasses of `ADemoSplineCraftActor` and asset references keep working as-is.
- Only the plugin folder name (`SplineCraftPRO` → `SplineCraftDemo`) and the `FriendlyName`
  changed.
- The `BP_SplineCraft_PRO` asset was deliberately **not** renamed, so references in projects
  do not break.

### Engine support

- The plugin was updated for **Unreal Engine 5.8**.

### New

- **Root Mobility.** A new property on the actor, next to **Mobility**. It sets the mobility
  of the actor's own spline root, independently of the generated meshes. Default `Static` —
  older scenes are unchanged. Set it to `Movable` to attach a movement component (e.g.
  *Rotating Movement*) or to move / rotate the whole structure at runtime; set **Mobility**
  to `Movable` as well so the meshes follow (a generated mesh is now automatically bumped so
  it is never left *more static* than the root, which would abort the attach).

- **Build Scenario — building step by step.** A new **SplineCraft Demo Build Scenario** asset: an
  ordered list of steps, each step an existing preset + a delay + an offset + an appear mode
  (Instant / Rise From Below / Scale Up / Material Progress). An actor with a scenario
  assigned becomes a "director": on the `Play Build Scenario` button it spawns child
  SplineCraft Demo actors one by one along its own spline, each with its own offset — so one
  preset raised on Z gives the floors of a building. `Reverse` is the same sequence taking it
  back down. In the editor there is a **Preview Progress** slider (0→1) with no timers. Fully
  additive: without a scenario assigned the actor's behaviour is unchanged.

- **A `Content/Samples` folder of examples.** The plugin now ships a folder of ready-made
  assets: the `DA_Preset_IronFence` preset, the `DA_Preset_Tower_*` set and the
  `DA_Scenario_GrowingTower` scenario (a hexagonal tower that grows from the ground up). Use
  them as-is via **Apply Preset** / **Build Scenario**, or as a starting point for your own
  styles.

- **Style presets.** A new **SplineCraft Demo Preset** asset type stores a whole structure setup:
  mode, Posts, Tubes, Sections, Knobs, Free Knobs, Polygons, collision, the seed and Mobility.
  The actor gains a **Preset** field and two buttons — **Apply Preset** and **Save To
  Preset**. This lets you build a library of styles and move them between levels and
  projects. A preset deliberately **does not** touch the spline shape, the alignment tools,
  Spline Close Loop, the openings or Design Notes. Ready-made samples are in `Content/Samples`.

- **Openings — gates, wickets, passages.** A new **OPENINGS** section: a list of openings,
  each defined by a **start and width in centimetres along the spline** plus checkboxes for
  which elements it touches (Posts / Knobs / Sections / Tubes). Points are removed if they
  fall strictly inside an opening; sections and tubes if they overlap it. An element exactly
  on the border stays, so the gate posts on both sides survive. Works in Line and Curve. In
  the editor every opening is highlighted with an orange frame in the viewport (the
  **Show Openings In Viewport** switch; never drawn in game). By default the list is empty —
  existing scenes are unchanged.

- **Random Seed — reproducible randomization.** The **Randomization** section gains a
  **Use Random Seed** checkbox and a **Random Seed** field: with one seed the structure is
  built identically every time, and two identically configured actors look identical. The
  **Randomize Seed** button picks a new variation. The seed covers everything: scale,
  paddings, shift, rotation, Flip and the random mesh choice. By default the checkbox is off —
  existing scenes behave as before.

- **Collision settings.** A new **Collision** section: a collision profile, hit- and
  overlap-event generation, and a separate toggle for building polygon collision. The
  settings apply to **every** generated component alike — instanced meshes, spline meshes and
  polygons. Previously collision could not be configured at all: spline meshes had it
  hard-wired to Query Only, and instanced meshes were left at the engine default. Along with
  this, the collision events `OnSplineCraft{Section,Tube,Post,Knob}HitEvent` **started
  working** for the first time — they are enabled by the **Generate Hit Events** checkbox
  (which needs a profile with physics collision, such as `BlockAll`). Turned on with the
  **Override Collision** flag; while it is off the behaviour is unchanged.

- **Component counter in Details.** A new read-only **Stats** section shows what the last
  build cost: how many instanced components, how many instances in total, how many spline
  meshes (their count climbs fast in Curve mode) and how many procedural meshes for polygons.

- **Four new point-alignment tools.** Added alongside the existing Ellipse, Rectangle, Line,
  Sinusoid, Uniform and Bind To Surface:
  - **Arc** — part of an ellipse: set the start angle and the sweep;
  - **Regular Polygon** — a closed regular polygon with equal sides; the start angle rotates
    the shape in place, the Sharp Corners checkbox keeps the sides straight;
  - **Spiral** — a spiral, and a helix once it has height: the diameter changes from start to
    end, the number of turns is fractional;
  - **Zigzag** — points alternate to either side of the X axis, sideways or vertically.

  For Arc and Spiral the points are spaced evenly along the curve by default (the **Even
  Spacing** checkbox). The Utils panel is cleaner: only the selected tool's options are
  shown, and the dropdown has tooltips.

- **Catenary — a hanging chain.** Another alignment tool: the curve a chain takes when it
  hangs freely. Set the **span** and the **sag depth**; the ends stay level with the actor
  and the middle drops by exactly the given amount. **A negative sag turns the curve over and
  arches it upwards** — the shape of a real arched bridge. The points are spaced evenly along
  the curve by default.

- **Follow Spline — laying out along another spline.** A new tool: point it at another actor
  with a `SplineComponent` (a road, a river, a cliff edge, a ready path) and the points are
  copied from it, so the fence/railing runs right along it. **Lateral Offset** moves them a
  set distance to the side, staying strictly parallel through corners. **Reverse Direction**
  flips the walk, **Match Reference Height** takes the height from the reference or keeps
  every point level with the actor. Each point's type and the closed-loop state are carried
  over from the reference.

- **Polygons along the curve.** A polygon gains a **Follow Curve** checkbox: previously an
  edge ran as a straight chord from one spline point to the next, so a curved spline gave a
  polygon with straight sides. Now the outline is sampled along the curve itself. The step is
  set by the **Curve Resolution** field (cm). Off by default — existing polygons are
  unchanged.

- **Arbitrary polygon shapes.** Polygon caps used to be triangulated with a fan, so only
  convex outlines closed correctly — on a concave one (L-shaped, star-shaped, comb) the cap
  overlapped itself. Ear clipping is now used: any non-self-intersecting outline is
  supported, in either winding direction and with the collinear points the alignment tools
  produce. A self-intersecting outline falls back to the previous fan.

### Fixed

- **Rectangle alignment tool — even spacing.** Points are now spread along the edges in
  proportion to edge length, so the spacing is as even as possible all the way round a
  non-square rectangle (before, every edge got the same number of points regardless of
  length). With **Bevel > 0** the tool no longer subtracts 4 from the point count on every
  rebuild — a bevelled rectangle used to lose points and collapse as you tweaked it.
- **Merge to Static Mesh** now also bakes the geometry of an active **Build Scenario
  preview** (the child step actors), so a structure designed as a growing scenario can be
  merged into one mesh by scrubbing Preview Progress up first. When there is genuinely
  nothing to bake, the action now reports it with a message instead of doing nothing
  silently.

- **Merge to Static Mesh lost polygons.** The merge only collected static-mesh components
  (posts, sections, tubes, knobs), and polygons are a procedural mesh of a different type
  that the filter discarded. Now each polygon is baked into a temporary static mesh before
  the merge and takes part in the combine alongside the rest.

- **The alignment tools overwrote manual point edits.** The selected tool rebuilt the spline
  on **any** change to the actor, including while a point was being dragged, so the edit
  disappeared under the cursor. Now a tool only runs when one of its own settings changes.
  The **Apply Point Layout** button lays the points out again when it is wanted.

- **Bind To Surface missed the ground.** The end of the trace ray was built by subtracting
  the point's own X and Y, so the ray leaned towards the world origin: the further the
  structure stood from that origin, the more the trace missed. Now the ray goes straight
  down, and the search distance up and down is set by separate fields.

- **Bind To Surface: "Vertical Points" turned points the wrong way.** The tangent was
  replaced with a forward constant, so every point aimed along local X regardless of which
  way the spline ran. Now the tangent is only laid flat, keeping its direction and length.

- **Horizontal Shift on corners.** A shifted row did not stay strictly parallel to the
  original: corners were pulled inwards and skewed towards the longer segment. The shift is
  now built as a parallel offset (a mitre). Applies to Posts, Knobs, Sections, Tubes,
  Polygons in Line mode; straight runs are unchanged.

- **Polygons: wrong lighting and junk geometry.** The normals array was twice as long as the
  vertex array, so the engine discarded it entirely and assigned every vertex an "up" normal —
  the side walls lit as if horizontal. On top of that each wall was added four times, which
  caused z-fighting. The geometry was rewritten: caps and walls now have their own vertices
  and normals, tangents were added, and the wall unwrap runs along the perimeter and up the
  height. The winding is derived from the signed area, so a clockwise polygon no longer comes
  out inside-out.

- **Polygons used another element's point-index map.** Polygons walk every spline point but
  used the set of active indexes left behind by the last Section/Tube — some points
  collapsed. A polygon now builds its own index map.

- **Closed spline: the first point was not treated as a corner.** At the seam of the loop the
  shift was computed as if for the end of an open spline.

- **Hit events did not fire for Tube / Post / Knob in Line mode.** The hit-event binding for
  Instanced Static Mesh components had an incorrect method pointer, so the
  `OnSplineCraftTubeHitEvent`, `OnSplineCraftPostHitEvent` and `OnSplineCraftKnobHitEvent`
  delegates were not bound. Fixed.

- **`FDemoSCPolygon::Material` had no initializer**, which caused a validation error on startup
  on UE 5.8. Added `= nullptr`.

