# 13 — UMG Visualization

*🇬🇧 English | [🇺🇦 Українська](../uk/13-UMG-Visualization.md)*

The `FuzzyLogicUMG` module draws membership functions and results into an `FPaintContext`. Call the nodes from your `UserWidget`'s **On Paint** event.

## Plot Area

`FFuzzyPlotArea` contains:

| Field | Value |
|---|---|
| `Origin` | Top-left corner in widget coordinates |
| `Size` | Width and height of the area |
| `Range` | Values of the horizontal axis |

A membership of `1` is at the top edge, `0` at the bottom.

## Blueprint Nodes

| Node | Draws |
|---|---|
| **Draw Fuzzy Set** | A single membership function |
| **Draw Fuzzy Variable** | All of a variable's sets, in palette colors |
| **Draw Fuzzy Surface** | The aggregated result surface |
| **Draw Fuzzy Value Marker** | A vertical line for a crisp value |
| **Draw Fuzzy Membership Marker** | A horizontal line for a degree |
| **Draw Fuzzy Plot Axes** | The baseline and left axis |
| **Project Fuzzy Point** | A point's coordinates for a custom label |
| **Get Default Fuzzy Palette** | Six contrasting colors |

## Plotting the Latest Decision

1. Call `Evaluate` on the component.
2. Get `Get Last Output Surface` for the desired output.
3. Pass the surface to `Draw Fuzzy Surface`.
4. Take the crisp number from `Result.Outputs` and draw it with `Draw Fuzzy Value Marker`.

This way, the UI shows the same discretized curve that defuzzification used.

## On Paint Example

In a Blueprint `On Paint` event:

```text
Make Fuzzy Plot Area
  Origin = (24, 24)
  Size   = (420, 180)
  Range  = OutputSurface.Range

Draw Fuzzy Plot Axes
Draw Fuzzy Surface
Draw Fuzzy Value Marker
```

For `Draw Fuzzy Set` and `Draw Fuzzy Variable`, a `Samples = 129` parameter is usually enough. Increase it for a very wide widget or steep functions.

## Custom Shapes

Drawing works through the base `FFuzzyMembershipFunction`, so a new C++ shape is rendered without separate drawing code.

## Weighted Average

This method doesn't use the aggregated surface to compute the number. If you're building explanatory UI, mark the curve as illustrative, just as the Data Asset editor does.
