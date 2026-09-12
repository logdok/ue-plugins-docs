# 02 — Quick Start

*🇬🇧 English | [🇺🇦 Українська](../uk/02-Quick-Start.md)*

This example turns the distance to a target into a smooth aggression level.

## 1. Create a Data Asset

In the Content Browser, choose **Add → Miscellaneous → Data Asset**, then the **Fuzzy System** class. Name the asset `DA_EnemyBehavior` and open it.

## 2. Add an Input

In `Inputs`, create an entry:

| Field | Value |
|---|---|
| `Name` | `Distance` |
| `Range.Min` | `0` |
| `Range.Max` | `2000` |
| `Default Value` | `1000` |

Add three sets:

| Set | Shape | Parameters |
|---|---|---|
| `Near` | Trapezoid | `0, 0, 300, 700` |
| `Mid` | Triangle | `400, 900, 1400` |
| `Far` | Trapezoid | `1100, 1500, 2000, 2000` |

## 3. Add an Output

In `Outputs`, create `Aggression` with range `0…1`. Add:

| Set | Shape | Parameters |
|---|---|---|
| `Cautious` | Triangle | `0, 0.15, 0.55` |
| `Watchful` | Triangle | `0.25, 0.5, 0.75` |
| `Aggressive` | Triangle | `0.5, 0.9, 1` |

## 4. Write the Rules

Add to `Rules`:

```text
IF Distance IS Near THEN Aggression IS Aggressive
IF Distance IS Mid THEN Aggression IS Watchful
IF Distance IS Far THEN Aggression IS Cautious
```

Save the asset. The `Compile Log` should report that the system compiled successfully.

## 5. Check the Result

On the **Inference** tab, change `Distance` in the left panel. The `Aggression` card on the right shows:

- the current crisp value;
- the output sets as faint colored lines;
- the aggregated area;
- a white marker for the defuzzified result.

The bottom panel shows which rules fired and with what strength. Preview values are temporary and don't change the `Default Value` in the Data Asset.

## 6. Use It in Blueprint

1. Add a **Fuzzy Logic** component to an actor.
2. Set `System Asset = DA_EnemyBehavior`.
3. Before making a decision, call `Set Input`:
   - `Variable Name = Distance`;
   - `Value = distance to target`.
4. Call `Evaluate Output`:
   - `Output Variable Name = Aggression`;
   - store the returned number.
5. Use the number for speed, attack weight, animation selection, or another smooth parameter.

`Set Input` persists the value between evaluations. If only the distance changed, you don't need to set the other inputs again.

## 7. Transfer the System via JSON

In the editor toolbar, click **Save To JSON**. To create another Data Asset from the same preset, open it and click **Load Preset**. Importing validates the entire document first; a failed load doesn't change the asset.

The next section explains the data model: [03 — Core Concepts](03-Core-Concepts.md).
