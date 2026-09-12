# 16 — Demo Content

*🇬🇧 English | [🇺🇦 Українська](../uk/16-Demo-Content.md)*

The plugin includes two ready-made maps. If the `Plugins` root isn't visible in the Content Browser, enable it first via the settings (gear) icon → **Show Plugin Content**, then navigate to **Plugins → Fuzzy Logic Content → Demo**:

- **FuzzyDroneArena** → `Demo/FuzzyDroneArena/Maps/FuzzyDroneArena`;
- **FuzzyTurretDefense** → `Demo/FuzzyTurretDefense/Maps/FuzzyTurretDefense`.

The data, Blueprints, materials, and maps live under `Plugins/FuzzyLogic/Content/Demo`. Their packages belong to the plugin, so there's no need to copy files into your project's `Content`.

## A Separate Demo Module

`FuzzyLogic` contains the fuzzy-inference core. The Drone Arena code is factored into a separate runtime module, `FuzzyLogicDemo`; it isn't a dependency of the core and doesn't change how systems, components, JSON, or UMG work.

The module is required by the Drone Arena and Turret Defense maps. In Turret Defense, the turret's gameplay behavior, targets, projectiles, and physical shell casings are all assembled in Blueprints; the module only adds the orbital controller and the live fuzzy-inference panel in the viewport. Shell casings eject from the gun's side port, react to physics, accumulate on the floor for up to 14 seconds, and then disappear automatically.

## The Fire Control Panel in Turret Defense

In the bottom-right corner, the map shows a visualization of the **Fire Control** system. It draws three input membership functions (`AimError`, `ThreatPriority`, `WeaponHeat`), the aggregated `FireAuthorization` surface, and orange markers for the current crisp values. What's displayed is exactly the surface the engine used for defuzzification.

- **HIDE / SHOW** buttons — show or hide the panel;
- **PAUSE / RESUME** button — stop or resume the simulation;
- `V`, `P`, or `Space` — the corresponding hotkeys.

The panel is enabled by default. For its graphs it uses the public `FuzzyLogicUMG` functions, so it also serves as a working example for your own HUD.

## Excluding the Demo from a Packaged Game

To keep the examples in the editor but not prepare them for the game, add this to your project's `Config/DefaultGame.ini`:

```ini
[/Script/UnrealEd.ProjectPackagingSettings]
+DirectoriesToNeverCook=(Path="/FuzzyLogic/Demo")
```

After this, Demo won't end up in the cooked build. The plugin and `FuzzyLogic` work as usual.

## Fully Excluding the Demo Code from the Plugin Build

This is only possible when building the plugin from source. In `FuzzyLogic.uplugin`, remove the `FuzzyLogicDemo` module entry, then don't add `Source/FuzzyLogicDemo` and `Content/Demo` to your package. The `FuzzyLogic` core doesn't reference this module and will keep building without it.
