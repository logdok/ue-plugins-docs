# 14 — Fuzzy Drone Arena Demo

*🇬🇧 English | [🇺🇦 Українська](../uk/14-Drone-Arena-Demo.md)*

The host project contains a vivid demo scene where nine drones evaluate a single shared fuzzy system every frame.

## How to Run It

Open the map:

```text
/FuzzyLogic/Demo/FuzzyDroneArena/Maps/FuzzyDroneArena
```

Press Play or Standalone Game. The camera orbits the arena, the central reactor pulses, and the drones change trajectory and color.

## Assets

| Asset | Purpose |
|---|---|
| `/FuzzyLogic/Demo/FuzzyDroneArena/Maps/FuzzyDroneArena` | The demo map |
| `/FuzzyLogic/Demo/FuzzyDroneArena/DA_DroneBehavior` | The shared `UFuzzySystemAsset` |
| `/FuzzyLogic/Demo/FuzzyDroneArena/Materials/M_DemoGlow` | A parametric emissive material |
| `/FuzzyLogic/Demo/FuzzyDroneArena/DroneBehavior.json` | The editable JSON source of the system |
| `Tools/create_fuzzy_drone_arena.py` | Reproducible creation of the assets and map |

## Inputs

- `Distance` — the drone's current orbit radius;
- `ReactorHeat` — the reactor's sinusoidal state `0…1`;
- `Integrity` — the individual drone's state `0…1`.

## Outputs

- `Aggression` changes the desired distance to the reactor and its color;
- `OrbitSpeed` controls angular speed and the rotors;
- `Altitude` determines orbit height.

The HUD is styled as a space instrument panel. The top-left panel names the inputs and outputs, the top-center legend explains the exact color scale from `Aggression = 0.0` to `Aggression = 1.0`, and the bottom-left panel briefly describes the scene. On the right, live telemetry for the first drone (`D-01`) is shown. A pulsing `TELEMETRY LOCK` frame follows this drone in view and shares the same color as the drone and the indicator in the right panel. The changing numbers demonstrate that the scene uses runtime inference, not pre-recorded animation.

## The Role of JSON

The script reads `DroneBehavior.json` via `FuzzyLogicStatics.load_fuzzy_system_from_json` and writes the structure into `DA_DroneBehavior`. Once the map is created, the drones use the Data Asset; the JSON isn't parsed every frame and isn't needed by the packaged scene.

You can also open `DA_DroneBehavior`, change the system in the editor, and click **Save To JSON** to manually sync the external preset.

## What to Try

- change the shape of `ReactorHeat.Hot` and see how altitude reacts;
- increase the weight of the low-integrity rule;
- switch `Centroid` to `Weighted Average` and compare the behavior;
- add an output for beam brightness;
- use one Data Asset for dozens of additional agents.

The demo classes belong to the host project and don't add to the plugin's runtime code that a buyer receives.
