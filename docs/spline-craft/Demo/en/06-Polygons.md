# Polygons

*🇬🇧 English | [🇺🇦 Українська](../uk/06-Polygons.md)*

> ⚠️ **This is the SplineCraft Demo (evaluation) edition.** Every feature works exactly like the full plugin; only *how much* you can build with it is capped, an on-screen watermark is shown, and it does nothing in Shipping builds. Full list: [Demo Limitations](00-Demo-Limitations.md).

The **POLYGONS** section generates a solid filled shape with a procedural mesh
(`UProceduralMeshComponent`): a floor, a ceiling, a platform, a courtyard lid, a water plane
inside a fence. The outline of the shape is all of the spline points.

## Polygon (`FDemoSCPolygon`)

| Parameter | Purpose |
|---|---|
| **Visible** | build the polygon. |
| **Distance From Bottom** | height of the bottom face above the spline (cm). |
| **Height** | thickness of the prism (cm). |
| **Shift** (`FDemoSCShift`) | Horizontal / Vertical offset (cm). |
| **Yaw Rotation (Z)** | rotate the polygon around the vertical (degrees). |
| **Follow Curve** | trace the curved outline of the spline instead of a straight chord between points. No visible effect where the points are straight (`Linear` type). |
| **Curve Resolution** | spacing of samples along the curve (cm) when **Follow Curve** is on. Smaller is smoother and heavier. |
| **Material** | the material. **Without a material the polygon is not built.** |

`FDemoSCPolygons` has a shared **Visible All** switch.

## Outline shape

- The caps (top and bottom) are triangulated by **ear clipping**: any non-self-intersecting
  outline is supported — convex, concave (L-shaped, star-shaped, comb), with collinear points
  produced by the alignment tools.
- A self-intersecting outline falls back to a simple fan triangulation (correct only for
  convex shapes).
- The winding is derived from the signed area, so a polygon drawn clockwise does not come out
  inside-out.
- The lateral shift (**Shift.Horizontal**) is applied as a parallel offset (a mitre).

## Collision

Polygon collision is enabled by the **Polygons Create Collision** checkbox in the
**Collision** section (which needs **Override Collision** turned on). For purely decorative
shapes it can be turned off. The rest of the collision settings apply to polygons the same as
to other components — see [chapter 10](10-Runtime-And-Blueprint-API.md).

## Performance

Each polygon is one procedural mesh component; the **Stats** section in Details shows how
many. Polygons are included in [Merge to Static Mesh](11-Merge-To-Static-Mesh.md): before the
merge each one is baked into a temporary static mesh.

## See also

- [Performance and Baking](13-Performance-And-Baking.md)
- [Merge to Static Mesh](11-Merge-To-Static-Mesh.md)
