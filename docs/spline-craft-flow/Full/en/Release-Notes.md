# Release Notes

*🇬🇧 English | [🇺🇦 Українська](../uk/Release-Notes.md)*

A short summary of each release: what's new, what's fixed, what to watch for when upgrading.
The newest release is at the top.

---

## 7.0.0 — Unreal Engine 5.8

### Engine Support

- The plugin has been updated for **Unreal Engine 5.8**.
- **Runtime data compatibility is preserved.** Existing levels and Blueprint subclasses of
  `ASplineCraftFlowActor` load with the same spline shape and generation configuration. One-shot editor
  commands have been deliberately removed from the actor's schema and moved to `SplineCraftFlowEditor`;
  details are in the upgrade section below.

### New

- **SplineCraft Flow Preset** — a reusable Core Data Asset holding a generation style: Mode,
  Static Meshes, Actors, Mobility, and advanced scale/orientation behavior. In Details you can create one
  unique `P_<ActorName>` per selected actor, apply assigned presets in batch, or safely update the shared
  asset with a confirmation. The spline shape and **Loop** aren't changed. See
  [section 13](13-Presets.md).
- **SplineCraft Flow Tools** — a single editor panel for presets, copying/aligning the spline, evenly
  redistributing points, and Merge. The reference spline is chosen at the level of an exact component, all
  commands support Undo, and safe operations support batch actor selection.
- **The construction scales along with the actor.** A new property, **Scale Structure With Actor**
  (enabled by default): actor scale scales the whole construction without changing the number and spacing
  of elements. Previously, scaling the actor along the spline produced more meshes, and they overlapped
  (in Curve mode they became flattened). For actors from scenes saved before 7.0, the property is disabled
  automatically, so old levels don't change. See [section 09](09-Transform-Mobility-And-Blueprint.md).
- **Copy Spline From Actor** — a new Blueprint function: copies another actor's spline without dialogs,
  including at runtime.
- **Build Elements Along Spline** can now be called from any Blueprint (previously only from actor
  subclasses).
- **Spline Geometry Edit Mode replaced Placement Strategy.** A new editor mode with a modifier stack
  (**Line**, **Ellipse**, **Rectangle**, **Regular Polygon**, **Spiral**, **Lemniscate**, **Road**, **Tilt**,
  **Distribute Evenly**, **Bind To Surface**, **Vertical Loop**, **Loop Track**, **Insert Loops**,
  **Roller Coaster**) — the single way to place points on a spline. Unlike a strategy applied once and
  forgotten, a step in the modifier stack is kept, reordered, and previewed — you can go back and adjust
  the shape without rebuilding it from scratch. The old system (`PlacementStrategyClass`, `USplineBasic`,
  and all standard strategies — Line, Rectangle, Ellipse, Spiral, Uniform, Torus Knot, Bind To Surface,
  Change Spline Points Rotation) has been fully removed, along with the corresponding Blueprint assets in
  Content.
- **Loop Track** — a ready-made toy-track-style loop: a closed rounded-rectangle return with up to four
  independent vertical loops (one per side — entry, right, far, and left), each with its own enable
  toggle, position along the side, radius, lateral divergence, and point count. Divergence moves the loop
  away from the ground track outward near its base, so it doesn't cross itself; all loops join the ground
  section without tangent breaks.
- **Insert Loops** — inserts one or more vertical loops into an already existing spline (a **Loops**
  array, each with its own **Position** from 0 to 1, radius, divergence, and point count), without
  touching the spline's shape outside the loops. Foolproofing: loops that would end up closer together
  along the spline than the sum of their radii are automatically spread apart, and if there isn't enough
  room, they're skipped with a warning in the Output Log.
- **Roller Coaster** — a ready-made closed roller-coaster track: two straight sections with two 180°
  turns, a lift hill, first drop, airtime hills that decay via **Hill Height Decay** (with random height
  variation via **Random Seed**), a flat braking section back to the station, and banked turns. Every
  joint in the profile is smooth — zero slope at every seam, with no jolts.
- **Road** builds a winding road or track from scratch — a closed loop or a long open route that wanders
  using a blend of two harmonics with a given **Random Seed** (the number of bends and the amplitude are
  separate parameters). A closed track can optionally cross itself (a "figure 8"); the height at the
  crossing rises smoothly so the two passes separate — the same trick as **Crossing Gap** in
  **Lemniscate**. It can optionally trace the result onto the terrain (**Snap To Surface**), as before.

### Improvements

- **Clean module separation.** The runtime module `SplineCraftFlow` no longer depends on `UnrealEd`,
  Slate, AssetTools, AssetRegistry, ContentBrowser, or MeshMergeUtilities. Dialogs, the Content Browser,
  tool transactions, and mesh merging live in `SplineCraftFlowEditor`; the packaged game only gets the
  Core API.
- **Faster build.** Instances are added to components in a single batch; the random number generator is
  no longer re-spun for every segment. The result with the same **Random Seed** is unchanged.
- **Fewer components.** In Line mode, an extra empty component is no longer created for every mesh entry.
- **Undo** for **Copy Spline Data from another Spline**, **Merge to Static Mesh** (actor spawning and
  removal), and spline alignment.
- **Merge to Static Mesh** reports when there's nothing to merge.
- **More accurate tooltips** in the Details panel, fixed field-locking conditions.
- Fewer log messages while working in the editor.

### Bug Fixes

- **Editor hang** with a zero placement step: a zero-length mesh, **Additional Scale X** = 0 in Curve
  mode, **Spacing** equal to minus the mesh length, or a very small **Gap between Pivot Points**. Such an
  entry is now skipped with a warning in the log.
- **Material loss:** if **Materials** had more elements than the mesh had slots, the extra elements were
  permanently removed from the settings. They're now kept and simply ignored.
- **Merge to Static Mesh overwrote assets:** assets got the name `SM_SM_<actor>`, and merging the same
  actor again overwrote the previous mesh. The name is now `SM_<actor>` and always unique.
- **Rebuilding removed your components:** ISM, Spline Mesh, and Child Actor components added in a
  Blueprint subclass or on an actor instance were destroyed. Now only generated ones are removed.
- **Risk of state corruption on Undo/Redo:** components of actors along the spline were recreated with the
  same name, and the engine replaced the old object "in place".
- **Meshes detached from the spline** if the spline component was moved or rotated inside the actor (for
  example, after **Copy Spline Data**): Line mode instances and actors weren't standing on the spline.
- **Meshes mixed up between entries:** entries with same-named meshes from different folders, or a mesh
  with no materials, could end up in the same component and overwrite each other's mesh and materials.
- **Slot Materials didn't work in Line mode.** They now work in both modes.
- **Copy Spline Data lost point types** (`Linear`, `Constant` became curves) and didn't carry over the
  spline's closedness.
- **Vertical Align** on a strictly vertical section of the spline produced a random rotation.
- **Double rebuild** after spline alignment and after changing **New Number of Spline Points**.
- **Unregistered "garbage" components** on hidden sections in Curve mode.
- **Meshes in Curve mode flipped on vertical sections of the spline** (for example, in a loop) — the
  orientation was derived from world "up" and got lost when the spline ran straight vertically. Now, the
  **Stable Curve Orientation** property, enabled by default, derives orientation from the spline's own
  quaternion frame. Actors saved before this version keep the old behavior — the look of existing levels
  doesn't change.

### What to Watch for When Upgrading

- The old editor-only fields `MergeSettings`, `bReplaceSplineActor`, `SourceActorWithSpline`,
  `AlignToBeginActor`, `NumberOfSplinePoints`, and `NewNumberOfSplinePoints` have been removed from the
  actor. Merge settings and folders are now personal editor settings; **Replace Source Actor** is
  deliberately never remembered. Copy/Align use the exact **Reference Spline**, and redistribution uses
  the temporary **Even Point Count** field in **SplineCraft Flow Tools**. This doesn't change the saved
  points or the generated look of old actors.
- If you **scaled** an actor in an old scene and want the new scaling behavior, enable
  **Scale Structure With Actor** manually. Actors created already in 7.0 (including ones spawned at
  runtime) use the new behavior.
- If you used **Slot Materials** in **Line** mode, they now take effect.
- **Placement Strategy has been replaced with Spline Geometry Edit Mode.** Strategies were only a tool for
  placing points on the spline — a one-shot command in Details that placed points and was then forgotten.
  The new editor now fills that role. The `USplineBasic` class and the standard strategies (Line,
  Rectangle, Ellipse, Spiral, Uniform, Torus Knot, Bind To Surface, Change Spline Points Rotation) have
  been removed from the plugin. This doesn't affect points already placed on existing levels — they're
  ordinary spline data, not a reference to a strategy class. If you had written your own Blueprint
  strategy based on these classes, port its logic to the new editor.

### For C++ Developers

- The new `USplineCraftFlowPreset` and `ASplineCraftFlowActor::ApplyPreset()` are available in
  Runtime/Blueprint. The low-level `CopySplineFromComponent`, `AlignSplineEndToStart`, and
  `RedistributeSplinePoints` show no UI; the Editor module wraps them in batch transactions.
- Pointer properties have been switched to `TObjectPtr`. Code reading `StaticMesh`, `Material`,
  `SplineComponent`, etc. keeps compiling; assigning `TArray<UMaterialInterface*>` to
  `Materials` needs to be changed to `TArray<TObjectPtr<UMaterialInterface>>`.
- `CoreUObject` and `Engine` have become public dependencies of the `SplineCraftFlow` module.
