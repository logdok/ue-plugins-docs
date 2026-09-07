# Spline Points and Alignment

*🇬🇧 English | [🇺🇦 Українська](../uk/02-Spline-Points-And-Alignment.md)*

The shape of the structure is set by the spline at the actor's root. Points can be edited by
hand or laid out with one of the tools (`SplinePointsAlignTool`).

## Editing by hand

Select the actor — the spline component becomes active in the viewport. From there it behaves
like any `USplineComponent`:

- **Alt-drag** a point to add a new one; **Delete** removes the selected one.
- Drag the tangent handles to change curvature.
- Each point has a **Spline Point Type** in the Details panel: `Curve` (smooth), `Linear`
  (corner), `Constant`.

If an alignment tool is selected it does **not** wipe hand edits: a tool only runs when its
own settings change (see "Non-sticky tools" below).

## Common parameters

| Parameter | Purpose |
|---|---|
| **Mode** (`ESplineCraftUseMode`) | `Concrete` / `Line` / `Curve` — the global placement strategy. See [Line vs Curve Mode](09-Line-vs-Curve-Mode.md). |
| **Visible Last Point Elements** | show the posts and knobs at the last point of an open spline (`bCloseSection`). |
| **Spline Close Loop** | close the spline into a loop. Unavailable for the Line, Sinusoid, Arc, Spiral, Zigzag and Catenary tools. |
| **Num Spline Points** | how many points the selected tool builds (for tools that build a shape from scratch). |
| **Design Notes** | a free-text note on the actor. |

## The Utils panel

The **Tool** dropdown only shows the options of the selected tool; the rest are hidden. Every
dropdown entry has a tooltip. The **Apply Point Layout** button lays the points out again
manually — for when you actually want it.

## Alignment tools (Tool)

### Manual
Points are placed by hand. No tool touches the spline.

### Line
A straight run of evenly spaced points. **Step** — spacing (m). Needs ≥ 2 points.

### Rectangle
A closed rectangle. **Clockwise**, **Rectangle Length (X)** / **Rectangle Width (Y)** (m),
**Bevel** — corner bevel (m). A point is always placed on every corner (4, or 8 with a bevel);
the rest are spread along the edges in proportion to edge length, so the spacing is as even as
possible all the way round even on a non-square rectangle. Needs ≥ 4 points (≥ 8 with a bevel —
the count is raised to that if it is lower).

### Ellipse
A closed ellipse, evenly divided into the requested number of points. **Clockwise**,
**Diameter X** / **Diameter Y** (m). Needs ≥ 3 points.

### Sinusoid
A sine wave along the X axis. **Amplituda** (m), **Frequency**. Optional random spread of the
amplitude (**Random Settings**), which can be turned off with **Ignore Random Settings**.

### Uniform
Redistributes the **points that already exist** evenly along the spline. **Use Every Nodal
Point** — how many nodes to align through; **Vertical Points** — stand the points upright.

### Bind To Surface
Drops the **points that already exist** onto the surface below them by tracing straight down.
**Vertical Points** — stand the points upright; **Additional Vertical Shift** (cm);
**Trace Distance Above** / **Trace Distance Below** — how far up and down to search for a
surface (cm). Keep "above" small so a bridge or roof overhead is not picked up instead.

### Arc
Part of an ellipse. **Diameter X** / **Diameter Y** (m), **Start Angle** — starting angle
(0 points along +X), **Sweep Angle** — how far it sweeps (90 = a quarter, 180 = a half),
**Clockwise**. **Even Spacing** (on by default) — equal steps along the curve; turn it off
to step by angle.

### Regular Polygon
A closed regular polygon with equal sides. **Diameter** (m), **Start Angle** — rotates the
shape in place (e.g. stand a hexagon on a flat side instead of a corner), **Sharp Corners** —
keep the sides straight (off — the shape rounds into a blob), **Clockwise**. Needs ≥ 3 points.

### Spiral
A spiral, and a helix once it has height. **Start Diameter** / **End Diameter** (m) — the
diameter changes from start to end; **Turns** — number of turns (fractions allowed);
**Height** — total rise (m; 0 = a flat spiral); **Clockwise**; **Even Spacing**.
Good for spiral stairs, ramps, wells.

### Zigzag
Points alternate to either side of the X axis. **Step** — step along X (m), **Amplitude** —
swing to the side (m), **Vertical** — swing up and down instead of sideways (a saw-tooth
profile), **Sharp Corners** — keep the corners sharp (off — you get a wave).

### Catenary
The curve a chain takes when it hangs freely. **Span** — distance between the two ends (m;
the ends stay level with the actor), **Sag** — how far the middle drops below its ends (m).
**A negative value turns the curve over and arches it upwards** — the shape of an arched
bridge. **Even Spacing** — equal steps along the curve (equal chain links); turn it off to
step by X. For chain fences between bollards, rope barriers, garlands, cables — and, with a
negative value, arched bridges and vaults.

### Follow Spline
Copies the points of another actor's spline — a road, a river, a cliff edge, a ready path —
so the structure runs along it. **Reference Actor** — an actor with a spline component (it
cannot follow itself); **Lateral Offset** — run this far to one side (m; positive is the
right-hand side looking along the reference); **Reverse Direction** — walk from end to start;
**Match Reference Height** — take height from the reference (ride its rises and dips) or keep
every point level with the actor. Each point's type (smooth/corner) and the closed-loop state
are carried over from the reference, so sharp road junctions stay sharp.

## About Even Spacing

Equal steps of **angle** are not equal steps of **distance** once the arc's two diameters
differ or a spiral's radius grows: without Even Spacing the posts bunch up where the curvature
is higher. On an elliptical arc the step spread drops from about 3× to 1.05×, on a spiral from
5.5× to 1.25×.

## Non-sticky tools

A tool rebuilds the spline **only** when one of its own settings changes — the tool itself,
`Num Spline Points`, or a field of its options struct. So: turn the parameters and the shape
updates live; drag a point and the edit survives. The **Apply Point Layout** button lays the
points out again when you want a clean start. Saved scenes do not change visually: the points
sit in the level, they are not recomputed on load.

## See also

- [Quick Start](01-Quick-Start.md)
- [Line vs Curve Mode](09-Line-vs-Curve-Mode.md)
