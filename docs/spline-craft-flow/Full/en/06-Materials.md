# Materials

*🇬🇧 English | [🇺🇦 Українська](../uk/06-Materials.md)*

By default meshes use the materials from their asset. A mesh entry has three override mechanisms that
work the same way in **Line** and **Curve** modes.

## 1. Materials + Apply Every Index

| Parameter | Purpose |
|---|---|
| **Materials** | materials by mesh slot index: element 0 — slot 0, and so on. An empty element leaves the mesh's material. Extra elements (more than there are slots) are ignored. |
| **Apply Every Index** | which segments to apply to: those whose ordinal number (from 1) is a multiple of the value. 1 — all of them, 2 — every second one, 3 — every third one. |

The remaining segments keep the mesh's materials — this is how you alternate, for example, every third
tile being a different color.

## 2. Materials Distance Ranges

A list of material sets for spline sections. Each entry (`FSPMaterialsDistanceRanges`):

| Parameter | Purpose |
|---|---|
| **Materials** | materials by slot index, as above. |
| **Distance Ranges** | sections the set applies to — see [Distance Ranges](05-Distance-Ranges.md). |
| **Apply Every Index** | which segments within the sections to apply to (number from 1, multiple of the value). |

For each segment, the **first matching** entry is used — the one whose segment number satisfies
**Apply Every Index** and whose reference point lies in at least one of its ranges. Its **Materials**
override mechanism 1. If the matched entry's **Materials** list is empty, mechanism 1 applies instead.

## 3. Slot Materials

A targeted override of a specific slot's material by name. Each entry (`FSPSlotMaterial`):

| Parameter | Purpose |
|---|---|
| **Material** | the material. |
| **Material Slot Name** | the mesh's slot name (as in the static mesh editor). |
| **Apply Every Index** | which segments to apply to (number from 1, multiple of the value). |
| **Distance Ranges** | sections the override applies to. **With no ranges the entry has no effect** — to apply to the whole spline, set, for example, **Begin Of Distance** = 0 and a large enough **Distance (m)**. |

Slot Materials apply **on top of** mechanisms 1 and 2. If several entries target the same slot, the lower
one in the list wins. A slot name that doesn't exist on the mesh is ignored.

## Application Order

1. The mesh's materials.
2. **Materials Distance Ranges** (first matching entry) — or, if none matches,
   **Materials** with **Apply Every Index**.
3. **Slot Materials** — on top, in order top to bottom.

## Line and Component Count

In **Line** mode, instances with different resulting materials end up in different
`InstancedStaticMeshComponent`s. Every new material combination is another component and a few more draw
calls. Alternating via **Apply Every Index** typically gives 2 components per entry.

## See Also

- [Distance Ranges and Hidden Ranges](05-Distance-Ranges.md)
- [Performance](11-Performance.md)
