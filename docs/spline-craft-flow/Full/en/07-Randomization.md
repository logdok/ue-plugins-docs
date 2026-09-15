# Randomization and Random Seed

*🇬🇧 English | [🇺🇦 Українська](../uk/07-Randomization.md)*

A mesh entry's **Randomization Settings** add controlled spread of scale, offset, and rotation between
segments. Each parameter is given as a **From**..**To** range; the value is drawn uniformly from that
range. Each group is enabled with its own checkbox.

## Scale

| Parameter | Purpose |
|---|---|
| **Use Scale Randomization** | enable scale spread. |
| **Uniform Scale** | one multiplier for all axes — the **XYZ Scale** range. Off — separate **X Scale**, **Y Scale**, **Z Scale**. |
| **Use Same Scale For Start And End** | in **Curve** mode: the same multiplier at the start and end of the segment. Off — the end gets its own random value, and the mesh smoothly changes thickness. |
| **Match End And Start Scale** | in **Curve** mode: the start of each segment takes the previous segment's end scale — transitions between segments without steps. |

The random multiplier is multiplied by the spline point's scale and by **Additional Scale**. Scale values
can't be negative.

## Offset

| Parameter | Purpose |
|---|---|
| **Use Offset Randomization** | enable offset spread. |
| **X Offset** / **Y Offset** / **Z Offset** | offset ranges, cm: X — along the spline, Y — sideways, Z — upward. Added to **Additional Offset**. |

In **Line** mode a random X offset shifts the instance along the straight direction; in **Curve** mode it
shifts the segment along the curve.

## Rotation

| Parameter | Purpose |
|---|---|
| **Use Rotation Randomization** | enable rotation spread. |
| **Roll** / **Pitch** / **Yaw** | angle ranges, degrees. |
| **Flip X** / **Flip Y** | **Line** only: random 0° or 180° rotation around the vertical axis; both checked — a random angle that's a multiple of 90°. Only works when **Use Rotation Randomization** is enabled. |

> In **Curve** mode **Roll** spins the mesh around the curve, and **Pitch** and **Yaw** tilt the
> segment's tangents; **Flip** has no effect.

## Random Seed

An entry's **Random Seed** (default 0) makes the spread reproducible: with the same seed the
construction builds identically on every rebuild, on any computer, and in the game.

- Each segment gets its own sequence of random numbers, depending on the seed and the segment number. So
  extending the spline adds new segments at the end without changing the random values of existing ones.
- Change **Random Seed** to get a different spread variant.
- Two entries with the same seed and the same settings get the same spread.

> **Note.** Enabling or disabling a randomization group changes the order of random values drawn within
> a segment, so the look may change even with the same seed — this is a property of the approach itself.

## When Recalculation Happens

- On every property change in the editor.
- At game start (`BeginPlay`).
- At runtime — when **Build Elements Along Spline** is called (see
  [section 09](09-Transform-Mobility-And-Blueprint.md)).

## See Also

- [Static Meshes](03-Static-Meshes.md)
- [Line and Curve Modes](04-Line-vs-Curve-Mode.md)
