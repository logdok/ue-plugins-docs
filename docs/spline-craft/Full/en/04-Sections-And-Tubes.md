# Sections & Tubes

*🇬🇧 English | [🇺🇦 Українська](../uk/04-Sections-And-Tubes.md)*

**Sections** and **Tubes** place a mesh in the gap between consecutive active spline points.
Both use the same configuration type (`FSCElement`); the only difference is intent and typical
meshes: sections are panels, infill, boards; tubes are railings, rails, crossbars.

## Element (`FSCElement`)

| Parameter | Purpose |
|---|---|
| **Distance From Bottom** | height of the element above the spline (cm). |
| **Visible All** | switch for the entry. |
| **Use Every Multiple Of Index** | repeat step: an element is built every N spline points. |
| **Main Static Mesh Configuration** | the main mesh configuration (below). |
| **Detailed Visibility Configuration** | per-entry visibility rules — see [Visibility Rules](07-Visibility-Rules.md). |
| **Alternate Meshes** | First / Last / Odd / Even — swap by span position. |
| **Special Meshes** | swap by index ranges (**Visible At Index Ranges**). |
| **Random Meshes** | `Use Random Meshes` + a list; the mesh is picked at random. |
| **Knobs** | knobs mounted on the element (below). |
| **Visible Knobs** | switch for all of the element's knobs. |
| **Ignore Alternate / Special Meshes** | turn off the corresponding swap mechanism. |

## Main Static Mesh Configuration (`FSCElementMeshConfig`)

| Parameter | Purpose |
|---|---|
| **Static Mesh** | the span mesh. |
| **Visible** | show the mesh. |
| **Scale** | **Thickness (Y)** and **Height (Z)** — weight coefficients of the original size. |
| **Paddings** | insets (`FMargin`, cm): Left / Right shorten the span along the spline, Top / Bottom trim its height. |
| **Shift** | **Horizontal (Y)** / **Vertical (Z)** offset (cm). |
| **Roll Angle (X)** | tilt around the element's axis (degrees). |
| **Mode** (`ESplineCraftMode`) | `Line` or `Curve` for this particular mesh — applied when the actor's global **Mode** is `Concrete`. See [Line vs Curve Mode](09-Line-vs-Curve-Mode.md). |
| **Ignore … Randomization Settings** | turn off scale / padding / translation / rotation randomization separately. |
| **Randomization Settings** (`FSCElementRandomSettings`) | spread — see [Randomization](08-Randomization.md). |

**How the mesh fills the gap.** In `Line` mode the mesh is stretched along its X axis to cover
the distance between the points (minus Left / Right paddings), and scaled on Z to account for
Top / Bottom paddings. In `Curve` mode the mesh bends along a `SplineMeshComponent` curve and
the offset is applied through its start/end offset.

## Knobs on the element (`FSCElementKnob`)

A decorative mesh or actor attached to the span at a weighted position:

| Parameter | Purpose |
|---|---|
| **Weight Horizontal Position** | position along the span, 0..1. |
| **Shift** (`FSCShift`) | Horizontal / Vertical offset (cm). |
| **Scale** | scale (`FVector`). |
| **Rotation** | an added rotation. |
| **Static Mesh** / **Displayed Object** / **Actor** | what to display (`Mesh` or `Actor`). |
| **Ignore … Randomization** / **Randomization Settings** | per-knob spread. |

Knobs are built in both modes — Line and Curve. When an element is hidden, its knobs are
hidden with it.

## Section-level visibility

`FSCSections` / `FSCTubes`:

- **Visible All Sections** / **Visible All Tubes** — switch for the whole section.
- **Visible All Knobs** — switch for knobs on every element.
- **All Sections/Tubes Visibility Configuration** — visibility rules for all elements at once.

## See also

- [Line vs Curve Mode](09-Line-vs-Curve-Mode.md)
- [Knobs and Free Knobs](05-Knobs-And-Free-Knobs.md)
- [Visibility Rules and Openings](07-Visibility-Rules.md)
