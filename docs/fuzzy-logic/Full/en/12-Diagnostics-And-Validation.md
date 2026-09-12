# 12 — Diagnostics And Validation

*🇬🇧 English | [🇺🇦 Українська](../uk/12-Diagnostics-And-Validation.md)*

## Message Levels

`FFuzzyDiagnostic` has a `Severity`, text, and, if applicable, a `RuleIndex`.

| Level | Meaning |
|---|---|
| `Info` | Useful information with no problem |
| `Warning` | The computation completed, but a fallback was used or something was skipped |
| `Error` | The system can't be considered correctly evaluated |

`ToString()` and the Blueprint node `Fuzzy Diagnostic To String` return ready-to-display text like `Error: unknown set (rule 3)`.

## Data Asset Validation

`UFuzzySystemAsset::IsDataValid` is integrated into Unreal's standard content validation. Errors and warnings appear in the Message Log during content validation. In code and Blueprint, use `Validate System`.

`Load Preset` performs validation before changing the asset. `Save To JSON` also doesn't export a system with errors.

## Common Errors

### Invalid Range

`Max` must exceed `Min`, and both bounds must be finite. Fix the range before tuning the shapes.

### Duplicate or Invalid Name

Variable and set names must be valid identifiers. Don't use spaces, hyphens, or the same name for an input and an output.

### Unknown Variable or Set in a Rule

Check the spelling and case of authored names. The premise must reference an input, the consequence an output.

### Mixed AND and OR

A single rule doesn't support both connectives. Split the logic into several rules.

### No Rule for an Output

Add at least one rule whose consequence writes to every output you need.

### No Rule Fired

This is a runtime warning. The system returns the midpoint of the output range. Check the gaps between input sets and the scenario on the Inference tab.

## Partial Preview

The editor can show results for rules that compiled successfully even when other rules have errors. Such a result is marked as a partial preview. It's needed for diagnostics, but `bSuccess` remains `false`.

## Runtime Check

```cpp
const FFuzzyInferenceResult Result = FuzzyLogic->Evaluate();
if (!Result.bSuccess)
{
    for (const FFuzzyDiagnostic& Diagnostic : Result.Diagnostics)
    {
        UE_LOG(LogTemp, Warning, TEXT("%s"), *Diagnostic.ToString());
    }
    return;
}
```

If the system's structure is known and validated during authoring, you don't need a separate validation every frame. The component caches the compilation.

## What to Check Before Release

- all Data Assets pass content validation;
- edge and mid-range inputs have been checked in Inference;
- every output has rules;
- there are no unexpected fallback warnings;
- JSON presets round-trip correctly;
- game code checks `bSuccess` when the system could be external or variable.
