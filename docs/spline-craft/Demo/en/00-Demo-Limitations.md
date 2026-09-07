# Demo Limitations

*🇬🇧 English | [🇺🇦 Українська](../uk/00-Demo-Limitations.md)*

This is the **SplineCraft Demo** — the evaluation edition. It installs alongside the full
plugin so you can try every feature before buying. It is **not** a cut-down alternative:
nothing is stubbed or removed, only *how much* you can build in one place is capped.

## Capped

| | Demo | Full |
|---|---|---|
| Spline points per actor | **16** — layout tools clamp to it; points placed by hand past 16 are ignored at build time | unlimited |
| Generated components per actor | **250** — instanced meshes + spline meshes + polygon meshes combined; once reached, the remaining element layers are skipped for that actor | unlimited |
| On-screen watermark | shown while a level runs (`SplineCraft DEMO — points 16/16, meshes 220/250`) | none |
| Merge to Static Mesh | **disabled** — design, preview and iterate freely, but you cannot bake a permanent Static Mesh asset | available |
| Shipping builds | generates nothing, draws no watermark; the build still compiles and packages normally | ships with your game |

## Not capped — identical to the full plugin

- every placement mode: Concrete / Line / Curve;
- every spline-point alignment tool: Line, Rectangle, Ellipse, Regular Polygon, Arc, Spiral,
  Sinusoid, Zigzag, Catenary, Bind To Surface, Follow Spline, Uniform, Manual;
- Posts / Sections / Tubes / Knobs / Free Knobs / Polygons / Openings, with their full
  array / alternate / special / random mesh configuration;
- randomization with a fixed Random Seed;
- Presets and Build Scenarios;
- collision settings and the hit-event delegates.

Every other chapter of this guide applies to the demo unchanged — only the five rows above
differ.

## What the demo ships

- **`BP_SplineCraftDemo`** (`/SplineCraftDemo/Blueprint/`) — a ready-made actor, the demo
  counterpart of the full plugin's `BP_SplineCraft_PRO`. Drop it into a level, or place a
  **SplineCraft Demo Actor** from the *Place Actors* browser.
- The full primitive library — posts, sections, tubes, knobs — plus the colour materials and
  textures, under `/SplineCraftDemo/Primitives`, `.../Material`, `.../Material_Inst`,
  `.../Texture`.
- Sample content under `/SplineCraftDemo/Samples` — `DA_Preset_IronFence`, the
  `DA_Preset_Tower_*` set and `DA_Scenario_GrowingTower`.

## Both editions in one project

The demo (module `SplineCraftDemo`, `ADemoSplineCraftActor`, `FDemoSC*` types) and the full
plugin (module `SplineCraft`, `ASplineCraftActor`, `FSC*` types) are independent UObject types
and can be enabled together with no conflict — a demo actor and a full-plugin actor sitting
side by side in one level each behave on their own.

## Moving to the full edition

Presets and Build Scenarios you author with the demo are demo-typed assets
(`UDemoSplineCraftPreset`, `UDemoSplineCraftBuildScenario`) and are not read by the full
plugin — recreate them there after switching.

Full edition on Fab. Full documentation:
https://logdok.github.io/ue-plugins-docs/spline-craft/Full/en/
