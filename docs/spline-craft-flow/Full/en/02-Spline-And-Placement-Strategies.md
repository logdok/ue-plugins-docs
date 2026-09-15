# Spline and Spline Geometry Edit Mode

*🇬🇧 English | [🇺🇦 Українська](../uk/02-Spline-And-Placement-Strategies.md)*

The actor's `SplineComponent` defines the construction's shape. Points can be edited manually, built or
edited with a modifier stack in **Spline Geometry** editor mode, or copied from another spline.

## Manual Editing

Select the actor — the spline becomes active in the viewport. From there it works like any regular
`USplineComponent`:

- **Alt-drag** a point to add a new one; **Delete** removes the selected one.
- Drag the tangent handles to change curvature.
- Each point has a type: `Curve` (smooth), `Linear` (sharp), `Constant`.
- Point **Scale** affects mesh scale in that section, **Roll** tilts meshes around the spline.

The construction rebuilds immediately after every edit.

## Spline Parameters on the Actor

| Parameter | Purpose |
|---|---|
| **Loop** | close the spline into a loop. Control looping here specifically: the actor overwrites the component's flag on every rebuild. |

The current point count is shown by **Spline Geometry Edit Mode** and the **Spline Utilities** status
line. Even redistribution is done with the **Redistribute to N Points** button or the
**Distribute Evenly** modifier — utility numbers are no longer stored as actor properties.

## Spline Geometry Edit Mode

A dedicated editor mode for building the spline's shape with a stack of modifier steps — instead of
placing points by hand or with a one-shot command. A step in the stack isn't forgotten right away: it can
be disabled, reordered, duplicated, or removed, and you can see the result in the viewport before applying.

### How to Enter

1. Select one or more actors in the viewport that have at least one `Spline Component`.
2. Open the editor mode dropdown (the same one that has **Landscape**, **Foliage**, **Mesh Paint**) and
   choose **Spline Geometry**.

The mode panel has three parts.

**Target** — the list of found spline components (with their point counts), with a checkbox on each:
which splines take part in the preview and application. If multiple actors are selected, their splines
are grouped by actor name. You can build the same thing on several splines at once.

**Modifier Stack** — the stack itself, executed **top to bottom**. The **+ Add Modifier** button opens
the list of all available modifiers (generated automatically, so new types appear here on their own,
without a plugin update). Each step in the stack has:

| Element | Purpose |
|---|---|
| checkbox on the left | enable/disable the step without removing it from the stack. |
| name and label | **Generates points** — the step rebuilds the shape from scratch (replaces all points); **Modifies points** — edits existing points in place. |
| **Up** / **Down** | move the step within the stack. |
| **Dup** | duplicate the step along with its parameters. |
| **X** | remove the step from the stack. |
| expanded area | the parameters for that specific step — an ordinary Details panel. |

**Order matters.** A step that rebuilds the shape from scratch (**Generates points**) erases everything
that steps above it did. If such an enabled step isn't first but somewhere lower, all steps above it
become pointless — the panel immediately highlights them in orange and reads "No effect — overwritten by
a later Generator step below". Rule of thumb: one shape-generator step first (Line, Ellipse, Rectangle,
Regular Polygon, Spiral, Lemniscate, Vertical Loop, Loop Track, Roller Coaster, Road), then as many steps
as you like that edit existing points (Distribute Evenly, Tilt, Bind To Surface, Insert Loops).

At the bottom — **Live Preview** (draws the current stack result in the viewport in blue, without
touching the real spline), a summary "N spline(s) selected · M active step(s)", an **Apply to N Spline(s)**
button, and **Cancel** (clears the whole stack, the mode stays active).

### Applying

**Apply** runs the enabled stack steps in order on each checked spline and immediately rebuilds the
construction (`RerunConstructionScripts`) — all as a single Undo step. If a step rebuilt the shape from
scratch (**Generates points**) on a `SplineCraftFlow` actor's spline, that freshly generated section is
automatically switched to **Stable Curve Orientation** — even if the actor itself was loaded from an old
level in legacy orientation mode. This is new geometry, so there's no point following the old behavior;
the rest of the spline (and actors loaded without any changes) is unaffected.

## Modifiers That Rebuild the Shape from Scratch

Each of these steps fully replaces the spline's points — previous content doesn't matter. Marked in the
stack as **Generates points**.

### Line

A straight row of points.

| Parameter | Default | Purpose |
|---|---|---|
| **Direction** | (1, 0, 0) | direction of the line in the spline's local space; no need to normalize. |
| **Step** | 500 | distance between points, cm. |
| **Number Of Points** | 12 | number of points (minimum 2). |

The up vector stays perpendicular to the line even if it runs straight vertically.

### Ellipse

Points evenly spaced by angle on an ellipse centered at the actor's origin; the first point lies on the
+X axis, then goes clockwise or counterclockwise (**Clockwise**).

| Parameter | Default | Purpose |
|---|---|---|
| **Radius X** / **Radius Y** | 2600 / 1700 | radii along the X and Y axes, cm. |
| **Number Of Points** | 16 | number of points (minimum 3). |
| **Clockwise** | off | traverse clockwise instead of counterclockwise. |

To close the ellipse, enable **Loop** on the actor.

### Rectangle

A closed rectangle with rounded corners in the XY plane, points distributed in equal-length segments
along the straight sides and corner arcs.

| Parameter | Default | Purpose |
|---|---|---|
| **Half Length X** | 5000 | half the size along the local X axis, cm. |
| **Half Width Y** | 3000 | half the size along the local Y axis, cm. |
| **Corner Radius** | 1500 | radius of all four corners, cm; automatically clamped to half the shorter side. |
| **Point Count** | 64 | exact number of points (minimum 8). |
| **Clockwise** | off | traverse clockwise instead of counterclockwise. |

Closes the spline on its own (no need to enable **Loop**).

### Regular Polygon

A regular polygon with equal sides in the XY plane.

| Parameter | Default | Purpose |
|---|---|---|
| **Number Of Sides** | 6 | number of sides (and corners), minimum 3. |
| **Diameter** | 4000 | diameter of the circle the corners lie on, cm. |
| **Start Angle** | 0 | rotation of the shape, degrees; 0 puts a corner on the local +X axis. |
| **Sharp Corners** | on | sharp corners (`Linear`). Turn off for smoothed corners (`Curve`). |
| **Clockwise** | off | traverse clockwise instead of counterclockwise. |

Closes the spline on its own.

### Spiral

A spiral around the local Z axis; with nonzero height it becomes a helix.

| Parameter | Default | Purpose |
|---|---|---|
| **Start Diameter** / **End Diameter** | 700 / 4000 | diameter at the first and last point, cm; equal values give a constant-radius helix. |
| **Turns** | 3 | number of turns (fractional). |
| **Height** | 3000 | rise from the first to the last point, cm; 0 gives a flat spiral. |
| **Number Of Points** | 48 | number of points (minimum 3). |
| **Clockwise** | off | traverse clockwise instead of counterclockwise. |

Good for spiral staircases, ramps, switchbacks.

### Lemniscate

The lemniscate of Bernoulli — a "figure 8" in the XY plane that crosses itself at the actor's origin.

| Parameter | Default | Purpose |
|---|---|---|
| **Size** | 2000 | half the width of the shape, cm — distance from the center to the farthest point along the X axis. |
| **Crossing Gap** | 200 | vertical gap between the two passes through the self-crossing point, cm — so a track built along the curve doesn't cross itself there. It ramps smoothly from each outer end of the shape (where the curve is flat) up to half the gap above the center on one loop and half below on the other. 0 makes the whole shape flat. |
| **Number Of Points** | 38 | number of points along the curve (minimum 8). |
| **Clockwise** | off | traverse the shape in the opposite direction. |

### Vertical Loop

A run-up along the local X axis, a full vertical loop, and an exit run from the landing point — a
"loop-the-loop" style coaster element.

| Parameter | Default | Purpose |
|---|---|---|
| **Approach Length** | 2000 | length of the run-up before the loop, cm; 0 makes the loop start right at the beginning of the spline. |
| **Exit Length** | 2000 | length of the exit run after landing, cm. |

The run-up and exit are themselves broken into segments with a step close to the loop's step — the whole
spline comes out even right away, without a separate **Distribute Evenly**.
| **Loop Radius** | 800 | radius of the loop, cm. |
| **Loop Divergence** | 500 | lateral offset between the start and end of the loop, cm, ramping up evenly over the whole turn — the vertical loop follows a helix (corkscrew) rather than a flat circle. 0 makes the loop flat. |
| **Loop Point Count** | 24 | number of points on the loop's circle (minimum 8). |

If **Loop** is already enabled on the spline, after landing the loop gets a separate return segment back
to the starting point — otherwise the spline stays open and ends at the exit.

### Loop Track

A ready-made closed track: a ground return shaped like a rounded rectangle (Entry/Right/Far/Left — the
entry, right, far, and left sides) with up to four independent vertical loops, one per side. Each loop
joins the ground section without tangent breaks.

| Parameter | Default | Purpose |
|---|---|---|
| **Ground Half Width** | 3000 | half the width of the ground return across the entry direction, cm. |
| **Ground Half Length** | 5000 | half the length of the ground return along the entry direction, cm. |
| **Ground Corner Radius** | 1500 | radius of all four corners of the ground section, cm. |
| **Ground Point Count** | 36 | number of points in the ground section, distributed evenly by length (across all straights and rounded corners); independent of how many loops are enabled. |

Next come four identically structured groups — **Loop 1 (Entry Side)**, **Loop 2 (Right Side)**,
**Loop 3 (Far Side)**, **Loop 4 (Left Side)** — each with its own set of parameters:

| Parameter | Default | Purpose |
|---|---|---|
| **Enabled** | on for Entry only | enables the loop on that side. From one up to all four can be enabled at once; with none enabled you get a plain rounded rectangle. |
| **Position** | 0.5 | where exactly on that side the loop sits: 0 and 1 are closer to the edges (but not flush — at least one corner radius of straight section is kept, so as not to ruin the corner), 0.5 is exactly in the middle. |
| **Radius** | 1800 | radius of the vertical loop on that side, cm. |
| **Divergence** | 500 | lateral offset of the loop per turn, cm (see **Vertical Loop** above) — the loop follows a helix and immediately moves away from the ground section outward, instead of crossing itself. 0 makes the loop flat. |
| **Point Count** | 24 | number of points on that loop's circle (minimum 8). |

Enables **Loop** on the spline by itself. If you then disable **Loop** on the actor manually, only the
final segment that closes the path disappears.

### Roller Coaster

A ready-made closed roller-coaster track: two straight sections joined by two 180° turns, with a height
profile on top (lift hill → first drop → shrinking airtime hills → a flat braking section back to the
station). Every joint (lift-to-crest, drop-to-valley, hill-to-hill, turn-to-straight) is smooth — the
slope is zero at every seam, like transition curves on a real coaster lift, so there are no jolts. Turns
bank into the arc and level out smoothly on the straights.

| Parameter | Default | Purpose |
|---|---|---|
| **Straight Length** | 6000 | length of each of the two straight sections connecting the turns, cm. |
| **Turn Radius** | 2500 | radius of the two turns at the ends of the track, cm. |
| **Track Point Count** | 96 | number of points, distributed evenly around the whole loop by horizontal distance. |
| **Lift Hill Height** | 3500 | height of the lift hill's crest above the station, cm. |
| **Lift Hill Length** | 5000 | how much of the track's length the lift hill climb takes up, cm; smooth entry and exit. |
| **First Drop Length** | 3000 | how much of the track's length the drop after the crest takes up, cm; smooth entry and exit. |
| **Number Of Hills** | 3 | number of airtime hills after the first drop, taking up the rest of the loop. 0 gives a flat braking section instead. |
| **First Hill Height** | 1800 | height of the first hill right after the drop, cm. |
| **Hill Height Decay** | 0.7 | height of each subsequent hill relative to the previous one. 1 makes all hills the same height; less makes the hills decrease toward the station, the classic profile. |
| **Hill Height Variation** | 0.15 | randomly varies each hill's height within this fraction around its base (decaying) height — real coasters are rarely perfectly geometric. Each hill stays just as smooth, only the crest height changes. 0 keeps hills exactly on the calculated curve. |
| **Random Seed** | 0 | seed for the random hill height variation. The same seed always reproduces the same track; change it to get a different, equally correct variant. |
| **Max Bank Angle** | 25 | how much the turns bank into the arc, degrees. Smooth entry and exit on each turn; the straights stay level. |

### Road

Builds a winding road, track, or path from scratch: a closed loop or one long open route that wanders
sideways from its axis using a blend of two harmonics (not a plain sine wave) with a given seed — the
result looks organic rather than perfectly mathematical. A closed track can optionally cross itself
(like a figure-8 interchange); wherever that happens, the height rises smoothly so the two passes
separate — the same trick as **Crossing Gap** in **Lemniscate**. The result can optionally also be
traced onto the terrain beneath it.

| Parameter | Default | Purpose |
|---|---|---|
| **Closed Loop** | on | build a closed loop instead of one long open route. |
| **Base Radius** | 8000 | average radius of the closed track, cm, before adding the bends. Closed track only. |
| **Length** | 15000 | length of the open route along its axis, cm, before adding the bends. Open route only. |
| **Number Of Points** | 150 | number of points along the whole road. A denser step follows sharp bends and terrain bumps more accurately. |
| **Number Of Bends** | 5 | number of S-shaped bends in the road. 0 leaves a flat circle (closed) or a straight line (open). |
| **Bend Amplitude** | 1500 | how far the road strays sideways from its axis, cm. |
| **Random Seed** | 0 | seed for the road's bend pattern — the phase of the main bends and a smaller, secondary wobble on top of them for a more organic, less mathematically perfect look. The same seed always reproduces the same road. |
| **Allow Self Crossing** | off | allow a closed track to cross its own path (figure-8-style crossing) instead of always staying a simple loop. Only meaningful for a closed track — an open route (lateral offset from a straight axis) can't physically cross itself regardless of amplitude. |
| **Crossing Gap** | 400 | vertical gap at each self-crossing point, cm, so the two passes don't collide — like a short bridge over the other lane. Smooth entry and exit on both sides. |
| **Crossing Ramp Length** | 2000 | how much of the track's length the climb onto the bridge (and descent from it) takes up at each crossing point, on each side, cm. |
| **Snap To Surface** | off | drop each point straight down onto the terrain beneath it, so the road follows the terrain's bumps instead of staying flat. |
| **Follow Surface Slope** | on | tilt each point to follow the terrain's slope (along with the cross-slope). Turn off to keep all points pointing straight up regardless of the ground's tilt. |
| **Surface Vertical Shift** | 0 | additional vertical offset after landing on the terrain, cm. |
| **Surface Trace Distance Above** | 2000 | how high above the point to search for the terrain, cm. |
| **Surface Trace Distance Below** | 5000 | how far below the point to search for the terrain, cm. |

## Modifiers That Edit Existing Points

These steps don't build a shape from scratch — they need points that already exist on the spline (drawn
by hand or created by a generator step earlier in the same stack). Marked as **Modifies points**.

### Distribute Evenly

Redistributes the existing spline so points are spaced evenly by curve length.

| Parameter | Default | Purpose |
|---|---|---|
| **Number Of Points** | 28 | new number of points (from 2 to 64). |
| **Keep Rotation** | on | keep the spline's roll (up vector) at the new points; turn off to reset the roll. |

Point scale is preserved along the spline's profile, the type of all new points becomes `Curve`.

### Tilt

Adds roll to existing points without moving them or touching their tangents or type.

| Parameter | Default | Purpose |
|---|---|---|
| **Additional Roll** | 0 | additional roll, degrees. |
| **Distribute Across Points** | off | spread **Additional Roll** progressively along the spline: the point at index *i* (from zero) gets `Additional Roll × (i + 1) / N`, the last point gets the full angle. When off, every point gets the full angle at once. |

### Bind To Surface

Drops existing points onto the surface beneath them.

| Parameter | Default | Purpose |
|---|---|---|
| **Use Custom Point Count** | off | instead of the existing points, take a new count evenly redistributed along the spline's current shape before dropping onto the surface. |
| **Number Of Points** | 20 | new point count, if **Use Custom Point Count** is enabled. |
| **Additional Vertical Shift** | 0 | additional vertical offset from the found surface, cm. |
| **Vertical Points** | on | each point's up vector points straight up, regardless of the surface's tilt. Turn off to have the up vector follow the surface normal at the hit point. |
| **Trace Distance Above** | 500 | how high above the point to search for the surface, cm. Keep this small if there's a bridge or roof above the spline. |
| **Trace Distance Below** | 2000 | how far below the point to search for the surface, cm. |

The trace goes straight down in world space, on the `WorldStatic` channel; the actor itself is ignored
(other objects, including ones attached to the actor, are not). If no surface is found, the point stays
in place.

### Insert Loops

Inserts one or more vertical loops into an already existing spline (drawn by hand or built by a previous
generator step) at specified locations, instead of building a whole track from scratch. Everything
outside the loops keeps its original shape — a loop only adds new points and shifts the rest of the
spline sideways by exactly its **Divergence**.

| Parameter | Default | Purpose |
|---|---|---|
| **Loops** | empty | array of loops to insert; use the "+"/"−" buttons to add or remove loops. |

Each array element is a separate loop with its own parameters:

| Parameter | Default | Purpose |
|---|---|---|
| **Enabled** | on | inserts this loop. Turn off to keep the settings without inserting — more convenient than deleting and recreating the element. |
| **Position** | 0.5 | where along the whole spline to insert the loop: from the start (0) to the end (1); 0.5 is exactly in the middle. |
| **Radius** | 1800 | radius of the loop, cm. Also used to check for overlap with neighboring loops (see below). |
| **Divergence** | 500 | lateral offset of the loop per turn, cm (see **Vertical Loop** above) — the loop follows a helix and immediately moves away from its own base instead of crossing itself. Shifts the rest of the spline after the loop sideways by that same distance. 0 makes the loop flat, with no offset. |
| **Point Count** | 24 | number of points on that loop's circle (minimum 8). |

Each loop's plane always contains the local vertical and the spline's horizontal travel direction at the
insertion point — a "vertical loop" stays vertical regardless of how much the track climbs or tilts there.

**Foolproofing.** If two loops end up closer along the spline than the sum of their radii (i.e. their
"bubbles" would overlap), the later one is automatically pushed further along the spline exactly far
enough to clear its neighbor. If there isn't enough room left before the end of the spline for this — or,
for a closed spline, room to also clear the first loop after closing — that loop is skipped entirely,
with a warning in the Output Log (`LogSplineCraftFlow`), and the rest of the loops are inserted as usual.

## Quick Spline Tools

One-shot commands are grouped under **Details → SplineCraft Flow Tools → Spline Utilities**. Their inputs
are temporary and aren't serialized into the runtime actor.

- In **Reference Spline** pick an exact `actor / Spline Component` pair, then click **Align End to
  Start**. The last point of each selected spline will match the first point of the reference spline in
  position and rotation, in world space. This is more precise than the old `AlignToBeginActor` field,
  which silently took the actor's first component.
- Set **Even Point Count** and click **Redistribute to N Points** to evenly resample the current shape.
  For a closed spline, the first point isn't duplicated at the seam. The old `NumberOfSplinePoints` and
  `NewNumberOfSplinePoints` fields are no longer stored on the actor.

Both commands work in batch on multiple selected actors and are undone with a single **Undo**.

## Copying Data from Another Spline

This is a separate editor tool in the **Details** panel. It doesn't store a temporary reference to the
source in the actor, and lets you explicitly pick the right component if one actor contains several
splines.

1. Select one or more SplineCraftFlow actors in the scene.
2. In the expanded **SplineCraft Flow Tools → Spline Utilities** category, click
   **Reference Spline**.
3. From the list pick the exact `actor / Spline Component` pair. Actors without splines, the selected
   target actors, and objects from another editor world are filtered out automatically.
4. Check the summary: point count, whether the spline is open or closed, and how many target actors.
5. Click **Copy Spline** for a single actor, or **Copy to N** for a batch copy.

All points are copied to every selected actor along with their tangents, types, roll, scale, local up
vector, and closedness (**Loop** is taken from the source). Each target actor is moved so its spline
exactly matches the source in world space, after which the construction rebuilds. The whole batch copy is
undone with a single **Undo**.

The **Reference Spline** selection is temporary: it belongs only to the current Details panel, isn't
serialized into the actor, and doesn't reach the runtime build. If the source is deleted, moved to
another world, or becomes one of the target actors, the button is disabled and the panel explains why.

From Blueprint, the **Copy Spline From Actor** function is available — it copies the first found
`Spline Component` of an actor without any dialogs and works at runtime too. When the source has multiple
splines and you need a specific component, use the editor picker above instead. More on the Blueprint API
in [section 09](09-Transform-Mobility-And-Blueprint.md).

## See Also

- [Quick Start](01-Quick-Start.md)
- [Static Meshes](03-Static-Meshes.md)
- [Transform, Mobility, and Blueprint API](09-Transform-Mobility-And-Blueprint.md)
