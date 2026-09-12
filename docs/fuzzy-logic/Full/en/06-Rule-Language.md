# 06 — Rule Language

*🇬🇧 English | [🇺🇦 Українська](../uk/06-Rule-Language.md)*

Rules are stored as readable text and compiled into internal indices before evaluation.

## Grammar

```text
rule       := "IF" antecedent "THEN" clause [ "WITH" number ]
antecedent := clause { ("AND" | "OR") clause }
clause     := identifier "IS" [ "NOT" ] identifier
identifier := letter { letter | digit | "_" }
```

## Examples

```text
IF Temperature IS Cold THEN Fan IS Slow
IF Temperature IS Hot AND Humidity IS High THEN Fan IS Fast
IF Battery IS NOT Low OR Distance IS Near THEN Aggression IS Bold
IF Speed IS Extreme THEN Brake IS Hard WITH 0.8
```

## Names and Keywords

`IF`, `IS`, `NOT`, `AND`, `OR`, `THEN`, `WITH` are case-insensitive. Spaces, tabs, and line breaks are free-form. Variable and set names must match the system and are case-sensitive.

Premises refer only to inputs. The consequence refers only to an output. An unknown name, using an output in the premise, or using an input in the consequence produces a diagnostic error.

## AND

`AND` combines degrees via the chosen T-norm. With the default `Minimum`, the rule's strength equals its weakest condition:

```text
Distance.Near = 0.7
Integrity.High = 0.4
AND / Minimum = 0.4
```

`Product` multiplies the degrees: `0.7 × 0.4 = 0.28`.

## OR

`OR` uses an S-norm. `Maximum` takes the strongest condition. `Probabilistic Sum` computes `a + b - ab` and gives mutual reinforcement without exceeding 1.

## NOT

`NOT` complements a specific statement:

```text
IF Health IS NOT Low THEN Speed IS Fast
```

If `Health.Low = 0.25`, then `Health NOT Low = 0.75`. `NOT` is also allowed in the consequence, though a separate, positively named output set is usually more readable.

## WITH

`WITH` sets a rule weight from `0` to `1`:

```text
IF Visibility IS Poor THEN Caution IS High WITH 0.65
```

The weight multiplies the premise's strength. An absent weight equals `1`.

## Single-Connective Constraint

A single rule uses either `AND` or `OR` throughout its premise. A mixed expression is rejected so that hidden operator precedence doesn't silently change intent:

```text
# Invalid
IF A IS High AND B IS Low OR C IS Near THEN Result IS Strong
```

Split it into two explicit rules, or introduce a separate variable if more complex logic is needed.

## Tips

- One rule should express one clear idea.
- Use weights for fine-tuning after the set shapes are already correct.
- Don't duplicate fully identical rules: under summing aggregation they can reinforce each other.
- The index in `RuleActivations` corresponds to the rule's position in the authored array, even when the rule failed to compile.

The mathematical behavior of the operators is explained in [07 — Inference And Settings](07-Inference-And-Settings.md).
