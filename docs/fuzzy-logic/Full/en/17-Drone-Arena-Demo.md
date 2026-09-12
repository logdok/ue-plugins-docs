# 17 — Fuzzy Drone Arena Demo

*🇬🇧 English | [🇺🇦 Українська](../uk/17-Drone-Arena-Demo.md)*

The host project contains a vivid demo scene where nine drones evaluate a single shared Mamdani fuzzy system every frame. There's no scripted choreography: each drone feeds its own orbit distance, the reactor's shared heat, and its own simulated integrity into the same `UFuzzySystemAsset`, and the resulting numbers drive its speed, altitude, and color in real time.

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

## How to Read the System

Unlike the Turret Defense demo's three chained systems, Drone Behavior is a single Mamdani system with three inputs and three outputs, evaluated independently by every drone. It uses the plugin's defaults throughout: `Minimum` for `AND`, `Maximum` for `OR`, `Clip` implication, `Maximum` aggregation, and `Centroid` defuzzification over 201 samples. `Trapezoid` and `Triangle` sets model the ranged inputs and outputs; `Ramp` models the two open-ended `0…1` inputs, where a falling ramp (`Shoulder` below `Foot`) reads high at the low end and a rising ramp reads high at the high end — see [Membership Functions](04-Membership-Functions.md).

## Drone Behavior — the Fuzzy System

| Variable | Range | Default | Set | Parameters | Meaning |
|---|---:|---:|---|---|---|
| `Distance` | 350…1500 | 900 | `Near` | Trapezoid: 350 / 350 / 520 / 850 | hugging the reactor; full membership up to 520, fading out to 850 |
| | | | `Mid` | Triangle: 600 / 900 / 1200 | working orbit; most typical around 900 |
| | | | `Far` | Trapezoid: 950 / 1250 / 1500 / 1500 | outer orbit; full membership from 1250 |
| `ReactorHeat` | 0…1 | 0.5 | `Cool` | Ramp: 0.75 → 0.15 | reactor running cold; full membership near 0.15 and below, fading out to 0.75 |
| | | | `Hot` | Ramp: 0.25 → 0.85 | reactor overheating; strength rises after 0.25 and is full near 0.85 |
| `Integrity` | 0…1 | 0.8 | `Low` | Ramp: 0.65 → 0.10 | damaged; full membership near 0.10 and below, fading out to 0.65 |
| | | | `High` | Ramp: 0.35 → 0.90 | healthy; strength rises after 0.35 and is full near 0.90 |
| `Aggression` | 0…1 | 0.5 | `Cautious` | Triangle: 0 / 0.18 / 0.55 | hang back; strongest near 0.18 |
| | | | `Watchful` | Triangle: 0.25 / 0.52 / 0.78 | hold position and observe; strongest near 0.52 |
| | | | `Aggressive` | Triangle: 0.55 / 0.88 / 1 | close in; strongest near 0.88 |
| `OrbitSpeed` | 0…1 | 0.5 | `Slow` | Triangle: 0 / 0.15 / 0.52 | drift; strongest near 0.15 |
| | | | `Fast` | Triangle: 0.42 / 0.88 / 1 | sprint; strongest near 0.88 |
| `Altitude` | 0…1 | 0.5 | `Low` | Triangle: 0 / 0.18 / 0.58 | skim near the platform; strongest near 0.18 |
| | | | `High` | Triangle: 0.42 / 0.82 / 1 | climb well above the reactor; strongest near 0.82 |

`Distance` closes a loop with the drone's own motion: every frame it is set to the drone's current orbit radius, clamped to the variable's range, and the resulting `Aggression` then nudges that same radius for the next frame (see below). `ReactorHeat` and `Integrity` are simulated signals — a shared heat wave for the former, an independent phase-shifted oscillation per drone for the latter — standing in for whatever real telemetry a game would report.

### The Rules

```text
IF Distance IS Near AND ReactorHeat IS Hot THEN Aggression IS Aggressive
IF Distance IS Near AND Integrity IS Low THEN Aggression IS Cautious
IF Distance IS Mid AND Integrity IS High THEN Aggression IS Watchful
IF Distance IS Far THEN Aggression IS Watchful

IF Distance IS Near THEN OrbitSpeed IS Slow
IF Distance IS Mid THEN OrbitSpeed IS Fast
IF Distance IS Far THEN OrbitSpeed IS Fast

IF ReactorHeat IS Cool THEN Altitude IS Low
IF ReactorHeat IS Hot THEN Altitude IS High
IF Integrity IS Low THEN Altitude IS High WITH 0.75
```

`Aggression` balances opportunity against risk: being near a hot reactor pushes it up, but low integrity while near pulls it back down, and a mid-range drone only commits to `Watchful` while healthy. Distant drones are always `Watchful` — there's no rule for `Far AND *` that reaches `Aggressive` or `Cautious`, so a lone distant drone settles on the middle set almost regardless of its other inputs. `OrbitSpeed` depends on `Distance` alone: drones hugging the reactor slow down (to avoid overshooting it), while mid-range and far drones both move fast. `Altitude` tracks `ReactorHeat` directly, with damaged drones (`Integrity IS Low`) also climbing — at a reduced `WITH 0.75` weight, so heat still dominates altitude when the two disagree.

## From Crisp Outputs to Motion

The three crisp numbers are read once per frame and drive the drone's actual transform and materials directly — there's no intermediate state machine:

| Output | Drives | Formula |
|---|---|---|
| `Aggression` | Desired orbit radius | `Lerp(1180, 560, Aggression)` cm, eased toward with `FInterpTo` |
| | Body color | see "What the Drone Color Means" below |
| | Point light intensity | `Lerp(420, 1050, Aggression)` |
| | Body glow strength | `Lerp(0.22, 0.58, Aggression)` |
| | Energy beam thickness | `0.009 + Aggression * 0.013` |
| `OrbitSpeed` | Angular speed | `Lerp(10, 38, OrbitSpeed)` degrees/second |
| | Rotor spin rate | `Lerp(180, 660, OrbitSpeed)` and `Lerp(240, 820, OrbitSpeed)` degrees/second, counter-rotating |
| `Altitude` | Orbit height | `Lerp(290, 820, Altitude)` cm, plus a small per-drone sine wobble |

The fed-back `Distance` input closes the loop described above: a high `Aggression` pulls the desired radius down to 560, which lowers `Distance` next frame, which the rules read as `Near` — reinforcing the same behavior until `Integrity` or `ReactorHeat` pulls the other way.

## What the Drone Color Means

The body color is not the fuzzy system's own output — it's a separate two-stage HSV blend over the same `Aggression` value, independent of where `Aggression`'s own `Cautious`/`Watchful`/`Aggressive` sets peak:

- `0.0 → 0.5`: blends from **Calm** (cyan, `0.015, 0.55, 1.0`) to **Adaptive** (violet, `0.42, 0.08, 1.0`).
- `0.5 → 1.0`: blends from **Adaptive** to **Approach** (red, `1.0, 0.015, 0.04`).

The HUD's legend swatches (`0.0 CALM`, `0.5 ADAPT`, `1.0 APPROACH`) mark this exact split — a deliberate, hardcoded visualization choice, not a boundary the fuzzy system itself defines. On the right of the HUD, live telemetry for the first drone (`D-01`) shows its `Distance`/`ReactorHeat`/`Integrity` inputs and `Aggression`/`OrbitSpeed`/`Altitude` outputs updating every frame; a pulsing `TELEMETRY LOCK` frame follows that drone in view, sharing its current color. The changing numbers demonstrate that the scene uses runtime inference, not pre-recorded animation.

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

## The Meaning of `DefaultValue`

`DefaultValue` on an **input** only matters before a drone's first `Tick` and as the starting point for the editor's Inference preview — at runtime, `Set Input` overwrites `Distance`, `ReactorHeat`, and `Integrity` from live simulation state every single frame, so the asset's defaults are never actually evaluated in the running demo.

`DefaultValue` on an **output** doesn't participate in inference at all. It's just the default value in the variable's structure; the result is always computed from the rules and membership functions.
