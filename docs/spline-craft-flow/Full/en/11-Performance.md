# Performance

*🇬🇧 English | [🇺🇦 Українська](../uk/11-Performance.md)*

## What the Construction Costs

| What | Cost |
|---|---|
| An entry in **Line** mode | 1 `InstancedStaticMeshComponent` per unique material combination for that entry. Hundreds of instances — a few draw calls. |
| An entry in **Curve** mode | 1 `SplineMeshComponent` per **each** segment. 500 segments = 500 components. |
| An actor entry | 1 `ChildActorComponent` + 1 actor per position. |

Component count is the main factor in editor rebuild time and in-game frame cost.

## Tips

- **Line by default.** Only enable **Curve** where the mesh actually needs to bend.
- **Fewer material combinations.** Every new combination in **Line** mode is another component.
- **Large final constructions — merge.** [Merge to Static Mesh](10-Merge-To-Static-Mesh.md) + Nanite is
  almost always a better result than thousands of live components.
- **Split long constructions across several actors.** Rebuilding after each edit is faster, and clamping
  is more precise.
- **For shaping the layout,** temporarily set **Mode** = `Visible Only Spline` — the spline can be moved
  without rebuilding the meshes.

## Limits and Safeguards

- If the step along the spline (mesh length + **Spacing**, in Curve accounting for **Additional Scale X**)
  is smaller than 0.01 cm, the entry isn't placed, and a warning is written to the log. This used to be
  able to hang the editor.
- The same applies to the actor step **Gap between Pivot Points** being smaller than 0.01 cm (0 still
  means a single actor, as before).
- No more than 1,000,000 elements are placed per entry; the rest are dropped with a warning.
- Plugin messages are written to the `LogSplineCraftFlow` log category.

## Lighting

- **Mobility** = `Static` is needed for components to take part in static lighting bakes.
- Rebuilding at game start recreates the components. If the project uses baked lighting, check the result
  in a packaged game or use Merge to Static Mesh.

## See Also

- [Line and Curve Modes](04-Line-vs-Curve-Mode.md)
- [Merge to Static Mesh](10-Merge-To-Static-Mesh.md)
