# 18 — Fuzzy Turret Defense Demo

*🇬🇧 English | [🇺🇦 Українська](../uk/18-Turret-Defense-Demo.md)*

The host project contains a second demo scene: a turret defends itself against waves of enemy drones using three chained Mamdani systems instead of a single hand-tuned threshold. One system scores threats, the second aims, and the third decides when it's actually safe to fire — and none of them is a switch statement.

## How to Run It

Open the map:

```text
/FuzzyLogic/Demo/FuzzyTurretDefense/Maps/FuzzyTurretDefense
```

Press Play or Standalone Game. Enemy drones approach from the arena's edge; the turret tracks the nearest one, spins up, and fires when its fuzzy `FireAuthorization` output clears the threshold. Shell casings eject from the gun's side port, land on the floor under physics, and fade out after 14 seconds. A **Fire Control** visualization panel in the bottom-right corner plots the three input membership functions and the aggregated `FireAuthorization` surface live — see [16 — Demo Content](16-Demo-Content.md) for its `HIDE`/`SHOW` and `PAUSE`/`RESUME` controls.

## Assets

| Asset | Purpose |
|---|---|
| `/FuzzyLogic/Demo/FuzzyTurretDefense/Maps/FuzzyTurretDefense` | The demo map |
| `/FuzzyLogic/Demo/FuzzyTurretDefense/DA_RadarAssessment` | Threat-priority `UFuzzySystemAsset` |
| `/FuzzyLogic/Demo/FuzzyTurretDefense/DA_TrackingControl` | Aiming-speed `UFuzzySystemAsset` |
| `/FuzzyLogic/Demo/FuzzyTurretDefense/DA_FireControl` | Firing-authorization `UFuzzySystemAsset` |
| `/FuzzyLogic/Demo/FuzzyTurretDefense/RadarAssessment.json`, `TrackingControl.json`, `FireControl.json` | The editable JSON sources of the three systems |
| `/FuzzyLogic/Demo/FuzzyTurretDefense/Blueprints/BP_FuzzyTurret` | The turret actor: rotation, radar sampling, firing, heat buildup |
| `Tools/create_fuzzy_turret_defense.py` | Reproducible creation of the assets and map |

The set names in the JSON are kept in English so they're easy to search for in Blueprint, but their exact meaning is given below.

The output number isn't a switch. It's a defuzzified result within the stated range: rules activate fuzzy sets, they overlap one another, and the **Centroid** method returns their center of gravity. That's why the transition between states is smooth.

## How to Read the Parameters

For `Triangle`, the parameters `Left`, `Peak`, `Right` mean: the degree of membership is `0` at the left and right points and `1` at the peak. Between them, the value changes linearly. For `Trapezoid`, full membership lasts from `LeftShoulder` to `RightShoulder`. For `Ramp`, the value changes from `0` at `Foot` to `1` at `Shoulder`; if `Shoulder` is less than `Foot`, it's a falling set.

The overlap between neighboring sets is intentional: for example, a result can be partially `Standby` and partially `Authorize` at the same time. See [Membership Functions](04-Membership-Functions.md) for more on shapes, and [Rule Language](06-Rule-Language.md) and [Inference Settings](07-Inference-And-Settings.md) for rules and defuzzification.

## 1. RadarAssessment — Threat Assessment

This system determines how much priority the nearest hostile target deserves. `TargetDistance` is measured in Unreal units, `ClosingRate` in Unreal units per second. Each frame, the turret computes `ClosingRate` as `(previous distance − current distance) / Delta Seconds` and clamps the result to the `0…1200` range: moving away gives `0`, fast closing gives a higher value. The `ThreatPriority` result is passed to FireControl.

| Variable | Range | Set | Parameters | Meaning |
|---|---:|---|---|---|
| `TargetDistance` | 0…3200 | `Critical` | Ramp: 1600 → 100 | critically close; full membership near 100 and below, fading out to 1600 |
| | | `Tactical` | Triangle: 700 / 1700 / 2700 | working range; most typical around 1700 |
| | | `Distant` | Ramp: 1900 → 3200 | far target; strength rises after 1900 and is full near 3200 |
| `ClosingRate` | 0…1200 | `Slow` | Ramp: 650 → 0 | the target is closing slowly or barely closing at all |
| | | `Fast` | Ramp: 250 → 1100 | fast closing; full membership near 1100 |
| `ThreatPriority` | 0…1 | `Observe` | Triangle: 0 / 0.12 / 0.45 | observe, no high priority |
| | | `Track` | Triangle: 0.30 / 0.58 / 0.82 | track the target |
| | | `Engage` | Triangle: 0.68 / 0.93 / 1 | threat is sufficient to engage |

Rules raise the priority for a close or fast-closing target. A distant, slow target gives `Observe`, a distant fast one gives `Track`, and a critical or tactical fast one gives `Engage`.

## 2. TrackingControl — Aiming Speed

This system chooses the servo speed. `AimError` is the angular error between the gun's axis and the target, in degrees. The `TrackingSpeed` result is a rotation-speed multiplier, not an angle.

| Variable | Range | Set | Parameters | Meaning |
|---|---:|---|---|---|
| `TargetDistance` | 0…3200 | `Near` | Trapezoid: 0 / 0 / 650 / 1450 | close target |
| | | `Medium` | Triangle: 800 / 1650 / 2500 | medium range |
| | | `Far` | Trapezoid: 1900 / 2700 / 3200 / 3200 | far target |
| `AimError` | 0…180° | `Aligned` | Trapezoid: 0 / 0 / 3 / 13 | gun is on target: full match up to 3°, fading out to 13° |
| | | `Tracking` | Triangle: 5 / 35 / 85 | target is being tracked |
| | | `Lost` | Ramp: 45 → 150 | large error: target nearly or fully lost |
| `TrackingSpeed` | 0.8…7.5 | `Precise` | Triangle: 0.8 / 1.5 / 3.2 | slow, precise fine-tuning |
| | | `Responsive` | Triangle: 2.2 / 4.2 / 6.1 | normal tracking |
| | | `Rapid` | Triangle: 4.8 / 7.0 / 7.5 | fast search or interception |

The main rule here is simple: `Lost` gives `Rapid`, `Tracking` gives `Responsive`, `Aligned` gives `Precise`. An additional rule guarantees `Rapid` when the target is both far and lost.

## 3. FireControl — Firing Authorization

This system combines accuracy, threat priority, and thermal state. Its output, `FireAuthorization`, is a smooth firing-permission indicator from `0` to `1`. It doesn't replace mechanical constraints: the Blueprint also checks the reload state.

| Variable | Range | Set | Parameters | Meaning |
|---|---:|---|---|---|
| `AimError` | 0…180° | `Locked` | Trapezoid: 0 / 0 / 2.5 / 9 | a solid lock; full match up to 2.5°, fading out to 9° |
| | | `Marginal` | Triangle: 3 / 14 / 34 | acceptable but imprecise aim |
| | | `Unsafe` | Ramp: 14 → 75 | dangerous error; full membership from 75° |
| `ThreatPriority` | 0…1 | `Low` | Ramp: 0.70 → 0.05 | low priority; full membership near 0.05 and below |
| | | `High` | Ramp: 0.35 → 0.95 | high priority; full membership near 0.95 |
| `WeaponHeat` | 0…1 | `Safe` | Ramp: 0.82 → 0.05 | safe temperature; full membership near 0.05 |
| | | `Hot` | Ramp: 0.58 → 0.98 | overheating; full membership near 0.98 |
| `FireAuthorization` | 0…1 | `Inhibit` | Triangle: 0 / 0.08 / 0.42 | hold fire; strongest near 0.08 |
| | | `Standby` | Triangle: 0.30 / 0.55 / 0.78 | keep tracking and wait for better conditions; peaks near 0.55 |
| | | `Authorize` | Triangle: 0.68 / 0.94 / 1 | permission to fire; strongest near 0.94 |

### What the `FireAuthorization` Values Mean

- `0…0.42`: the `Inhibit` set is active; firing isn't allowed.
- roughly `0.30…0.78`: the `Standby` region; the system may still track the target, but conditions aren't sufficient to fire.
- `0.68…1`: `Authorize` is active; the closer the result to `0.94`, the stronger the combined case for firing.

In the demo Blueprint, firing is actually permitted by the check `FireAuthorization > 0.64` together with a zero `FireCooldown`. The threshold is lower than the start of `Authorize` (`0.68`) so that no abrupt switching occurs in the transition zone, while still ensuring low, indeterminate results can't trigger a shot.

Safety rules take priority: `Unsafe`, `Low`, or `Hot` lead to `Inhibit`. Only the combination of `Locked`, `High`, and `Safe` gives `Authorize`; `Marginal`, `High`, `Safe` gives an intermediate `Standby`.

## The Role of JSON

`create_fuzzy_turret_defense.py` reads `RadarAssessment.json`, `TrackingControl.json`, and `FireControl.json` via `FuzzyLogicStatics.load_fuzzy_system_from_json` and writes each structure into its matching Data Asset (`DA_RadarAssessment`, `DA_TrackingControl`, `DA_FireControl`). Once the map is created, `BP_FuzzyTurret` reads the three Data Assets; none of the JSON files is parsed at runtime or needed by a packaged build.

You can also open any of the three Data Assets, change the system in the editor, and click **Save To JSON** to manually sync the external preset.

## What to Try

- lower the `FireAuthorization > 0.64` gate in the Blueprint and watch the turret fire earlier, less certainly;
- widen `AimError.Locked` in FireControl and see how much sloppier aim the turret tolerates before it stops firing;
- change `WeaponHeat.Hot` so the turret needs to cool down sooner;
- add a fourth `ThreatPriority` set (e.g. `Critical`) and a matching rule in FireControl;
- switch `FireControl`'s defuzzification from `Centroid` to `Mean of Maxima` and compare how decisively it commits to `Authorize`.

The demo classes belong to the host project and don't add to the plugin's runtime code that a buyer receives.

## The Meaning of `DefaultValue`

`DefaultValue` on an **input** is the starting value before the first `Set Input`. For example, `AimError: 90` means that before receiving telemetry, the system considers the gun's axis to be off by 90°.

`DefaultValue` on an **output** doesn't participate in inference. It's just the default value in the variable's structure; the result is always computed from the rules and membership functions. For `FireAuthorization: 0`, it's also a safe initial state before the first evaluation.
