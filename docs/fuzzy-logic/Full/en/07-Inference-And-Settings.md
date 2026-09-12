# 07 — Inference And Settings

*🇬🇧 English | [🇺🇦 Українська](../uk/07-Inference-And-Settings.md)*

## Four Stages

1. **Fuzzification.** Each crisp input is evaluated against all of its sets.
2. **Rule evaluation.** The statements' degrees are combined, `NOT` and the `WITH` weight are applied.
3. **Implication and aggregation.** The rule's strength shapes part of the output set, and all parts are combined.
4. **Defuzzification.** The aggregated shape is converted into a crisp number.

## System Settings

| Setting | Options | Effect |
|---|---|---|
| **And Operator** | `Minimum`, `Product` | T-norm for `AND` |
| **Or Operator** | `Maximum`, `Probabilistic Sum` | S-norm for `OR` |
| **Implication** | `Clip`, `Scale` | Applying rule strength to the consequence |
| **Aggregation** | `Maximum`, `Probabilistic Sum`, `Bounded Sum` | Combining rule consequences |
| **Defuzzification Method** | six methods below | Converting the result into a number |
| **Sample Count** | `11…4001`, default `201` | Resolution of the output curve |

The defaults `Minimum`, `Maximum`, `Clip`, `Maximum`, `Centroid`, `201` correspond to a typical Mamdani system.

## T-Norms and S-Norms

`Minimum` makes the weakest condition the limiting factor. `Product` gradually reduces strength for every incomplete condition.

`Maximum` lets the strongest alternative determine `OR`. `Probabilistic Sum = a + b - ab` reinforces several partially true alternatives.

## Implication

- `Clip` cuts off the consequence's membership function at the height of the rule strength. This is classic Mamdani.
- `Scale` multiplies the entire function by the rule strength and preserves its shape. This approach is often called Larsen implication.

## Aggregation

- `Maximum` takes the largest activation at each point.
- `Probabilistic Sum` smoothly reinforces agreeing rules.
- `Bounded Sum` adds activations and caps the sum at one.

Under `Maximum`, duplicate rules don't increase the peak. Under the summing methods they do, so watch for repeats.

## Defuzzification Methods

| Method | Result | Typical use |
|---|---|---|
| **Centroid** | Center of gravity of the area under the curve | Smooth continuous control; the default choice |
| **Bisector** | The point that splits the area in half | Less sensitive to a long thin tail |
| **Mean of Maxima** | Average of all maximum points | Selecting the most-supported result |
| **Smallest of Maxima** | Smallest maximum point | Conservative tie-breaking |
| **Largest of Maxima** | Largest maximum point | Aggressive tie-breaking |
| **Weighted Average** | Weighted average of the consequences' representative values | The fastest method, especially for Singleton |

`Weighted Average` doesn't discretize the output surface and ignores `Sample Count`. In the editor, its aggregated curve is marked as illustrative.

## When No Rule Fires

When no rule activates a particular output, the system returns the midpoint of the range and adds a warning. This is a predictable fallback, but not a real decision. Check the coverage of the input sets and the presence of rules for the output.

## Sample Count

`201` suits most gameplay tasks. Increase the value when the output range is wide, the shapes are narrow, or higher numeric precision is needed. Decrease it only after profiling. Cost scales with the number of outputs, rules, and samples.
