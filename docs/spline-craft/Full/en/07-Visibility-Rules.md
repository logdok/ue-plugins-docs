# Visibility Rules and Openings

*🇬🇧 English | [🇺🇦 Українська](../uk/07-Visibility-Rules.md)*

There are two independent ways to remove part of a structure: **visibility rules by index**
(`FSCVisibleConfig`) and **openings by distance** (`FSCOpenings`).

---

## Part 1. Visibility rules (`FSCVisibleConfig`)

They control which instances of a primitive are shown, based on their placement index. They
apply at two levels:

1. **Section level** — `All Posts / Sections / Tubes / Knobs Visibility Configuration`.
2. **Entry level** — the `Detailed Visibility Configuration` of a specific post, section, tube
   or knob.

An instance is shown only if it **passes both levels**.

### Parameters

| Parameter | Effect |
|---|---|
| **Hide At Indexes** | hide by a list of indexes. |
| **Hide In Index Ranges** | hide by index ranges. |
| **Visible Every Multiple** | show only when the index is a multiple of the value (0 = off). |
| **Hide Every Multiple** | the opposite: hide when the index is a multiple of the value. |
| **Visible First** / **Visible Last** | show the first / last instance. |
| **Visible Only Extreme** | show only the first and the last, hide the rest. |
| **Visible Odd** / **Visible Even** | show odd / even instances. |
| **Design Notes** | a free-text note. |

### Order of evaluation

1. If the index is in **Hide At Indexes** or **Hide In Index Ranges** — hide.
2. Otherwise, if **Visible Every Multiple** is set — show only on multiple indexes.
3. Otherwise, if **Hide Every Multiple** is set — hide on multiple indexes.
4. Otherwise apply the extreme and parity rules: first → `Visible First`,
   last → `Visible Last`, in the middle → `Visible Only Extreme` hides everything,
   and without it `Visible Odd` / `Visible Even` apply.

`Visible Every Multiple` and `Hide Every Multiple` are mutually exclusive — turning one on
hides the other's field.

---

## Part 2. Openings

The **OPENINGS** section cuts gaps into the structure — gates, wickets, passages —
**by distance along the spline in centimetres**, rather than by index arithmetic.

### `FSCOpenings`

- **Use Openings** — master switch for every opening below.
- **Openings** — the list of openings.

### Opening (`FSCOpening`)

| Parameter | Purpose |
|---|---|
| **Enabled** | enable this opening. |
| **Start** | distance from the start of the spline to the start of the opening (cm). |
| **Width** | width of the opening along the spline (cm). |
| **Affect Posts / Knobs / Sections / Tubes** | which elements the opening touches. |
| **Design Notes** | a free-text note. |

### How it works

- **Points** (Posts, Knobs) are removed if they fall **strictly inside** the opening.
- **Sections and tubes** are removed if they **overlap** the opening.
- An element exactly on the border **stays** — that is what keeps the gate posts on both
  sides.
- A hidden element takes its knobs with it, in both modes — Line and Curve.
- Openings **do not cut geometry**: whole elements are hidden. For a clean gate edge, place
  spline points at its borders.
- **Free Knobs** and **Polygons** are not affected by openings.

### Editor highlight

Every enabled opening is drawn in the viewport as an orange frame, so the distances can be
judged by eye. Turned off with the **Show Openings In Viewport** checkbox on the actor. The
frame is never drawn in game.

By default the openings list is empty — existing scenes are unchanged.

## See also

- [Posts](03-Posts.md)
- [Sections & Tubes](04-Sections-And-Tubes.md)
