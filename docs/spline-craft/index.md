# SplineCraft

<!-- last-synced:start -->
<p style="text-align: right; font-size: .75rem; opacity: .7;"><em>Docs last synced: 2026-09-07 21:35 UTC</em></p>
<!-- last-synced:end -->

**SplineCraft 7.0.0**, for **Unreal Engine 5.8**. Rapid procedural structures along a spline —
fences, railings, balustrades, colonnades, arcades, barricades, bridges, roads. Everything is
configured on a single `ASplineCraftActor` in the **Details** panel, and the geometry rebuilds
automatically on every change. From Blueprint or C++.

The full guide is available in English and Ukrainian:

| | English | Українська | Covers |
|---|---|---|---|
| **Full** — the complete plugin | [Guide](Full/en/README.md) | [Посібник](Full/uk/README.md) | Spline points and the 13 alignment tools, Posts / Sections / Tubes / Knobs / Polygons, visibility rules and openings, randomization, Line vs Curve mode, collision and hit events, presets and build scenarios, Merge to Static Mesh, the runtime/Blueprint API |
| **Demo** — evaluation build | [Guide](Demo/en/README.md) | [Посібник](Demo/uk/README.md) | The same guide with demo-edition naming (`ADemoSplineCraftActor`, `FDemoSC*`, …), plus what the demo specifically limits |

Start with the [Guide](Full/en/README.md), or jump to the
[Release Notes](Full/en/Release-Notes.md) for what's new in the Unreal Engine 5.8 update.

## Which edition is this for?

Both editions expose an identical feature set — every chapter of the Full guide has a matching chapter in the Demo guide, just with `Demo`-prefixed type names. The only *behavioural* differences — up to 16 spline points and 250 generated components per actor, an on-screen watermark, Merge to Static Mesh disabled, and nothing generated in Shipping builds — are covered in the Demo guide's **[Demo Limitations](Demo/en/00-Demo-Limitations.md)** chapter (Ukrainian: [Обмеження демо](Demo/uk/00-Demo-Limitations.md)). If you're not sure which one you're using, check your project's `Plugins/` folder: `SplineCraft` is the full edition, `SplineCraftDemo` is the demo. Both can be installed in the same project at once, side by side, with no conflict.

## Note on the name

The plugin was previously published as **SplineCraft PRO**. It is now simply **SplineCraft**.
The rename does not affect backward compatibility — the module name, content mount point, class
paths and property layout are unchanged, so existing levels and Blueprints keep working. See
the [Release Notes](Full/en/Release-Notes.md).
