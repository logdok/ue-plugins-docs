# 11 — JSON Presets

*🇬🇧 English | [🇺🇦 Українська](../uk/11-JSON-Presets.md)*

JSON lets you store a system outside the binary Data Asset, review changes to the system in version control, generate presets with tools, and move them between projects.

## Data Asset → JSON

Open a `UFuzzySystemAsset` and click **Save To JSON**. The editor:

1. compiles and validates the current system;
2. shows errors and doesn't write an invalid document;
3. opens the system save dialog;
4. appends `.json` if no extension was given;
5. writes a canonical document with `SchemaVersion`.

## JSON → Data Asset

Click **Load Preset** and choose a `.json` file. Loading happens into a temporary structure. If the syntax, schema, or rules are invalid, the open Data Asset isn't changed. After success, the editor:

- replaces the system;
- refreshes Details, Variables, Inference, and Compile Log;
- marks the package as modified;
- creates an Undo transaction.

The path to an external JSON file is editor state, not a runtime property of the Data Asset. A packaged game uses the asset and doesn't depend on the loose file.

## Blueprint API

| Node | Parameters |
|---|---|
| **Save Fuzzy System To Json** | `System`, `Content Relative Path`, `out Error` |
| **Load Fuzzy System From Json** | `Content Relative Path`, `out System`, `out Error` |

Blueprint paths are given relative to the `Content` directory, e.g. `FuzzySystems/Driving.json`. Attempting to escape `Content` via `..` is rejected. The plugin's built-in presets are available via their full mount path, e.g. `/FuzzyLogic/Demo/FuzzyDroneArena/DroneBehavior.json`.

## C++ API

```cpp
#include "Serialization/FuzzySystemJson.h"

FString Json;
FText Error;
FFuzzySystemJson::ToJsonString(System, Json, Error);
FFuzzySystemJson::FromJsonString(Json, System, Error);

FFuzzySystemJson::SaveToFile(System, AbsolutePath, Error);
FFuzzySystemJson::LoadFromFile(AbsolutePath, System, Error);

FString Absolute;
FFuzzySystemJson::ResolveProjectRelativePath(
    TEXT("FuzzySystems/Driving.json"), Absolute, Error);
```

Every operation returns a `bool`; `OutError` explains a failure.

## Schema 2

```json
{
  "SchemaVersion": 2,
  "Settings": {
    "AndOperator": "Minimum",
    "OrOperator": "Maximum",
    "Implication": "Clip",
    "Aggregation": "Maximum",
    "DefuzzificationMethod": "Centroid",
    "SampleCount": 201
  },
  "Inputs": [
    {
      "Name": "Distance",
      "Range": { "Min": 0, "Max": 2000 },
      "DefaultValue": 1000,
      "Sets": [
        {
          "Name": "Near",
          "Function": {
            "Type": "Trapezoid",
            "LeftFoot": 0,
            "LeftShoulder": 0,
            "RightShoulder": 300,
            "RightFoot": 700
          }
        }
      ]
    }
  ],
  "Outputs": [
    {
      "Name": "Aggression",
      "Range": { "Min": 0, "Max": 1 },
      "Sets": [
        {
          "Name": "Bold",
          "Function": { "Type": "Triangle", "Left": 0.4, "Peak": 1, "Right": 1 }
        }
      ]
    }
  ],
  "Rules": [
    "IF Distance IS Near THEN Aggression IS Bold"
  ]
}
```

## Fields

- `SchemaVersion` — the format version. A document with a newer, unknown version is rejected.
- `Settings` — all math settings; missing fields get default values.
- `Inputs`, `Outputs` — arrays of variables.
- `Name` and `Range` — required for a variable.
- `DefaultValue` — defaults to `0`.
- `Sets` — an array of sets with `Name` and `Function`.
- `Rules` — an array of authored rule text.

`Function.Type` accepts `Triangle`, `Trapezoid`, `Gaussian`, `Bell`, `Sigmoid`, `Ramp`, `JShape`, `Singleton`, `Constant`, or a registered custom type. Every shape supports `"Inverted": true`.

An unknown type, or corrupted or truncated JSON, ends with a clean error without crashing the editor.
