# Generation Style Presets

*🇬🇧 English | [🇺🇦 Українська](../uk/13-Presets.md)*

`USplineCraftFlowPreset` is a Data Asset holding a reusable SplineCraft Flow generation style. One preset
can be applied to many actors with different spline shapes: for example, save one set of fence sections,
posts, and materials, then quickly assign it to different routes.

All commands live under **Details → SplineCraft Flow Tools → Preset**.

## What's Included in a Preset

| Data | Purpose |
|---|---|
| **Mode** | the global element placement mode. |
| **Static Meshes** | all `FSPMeshConfiguration` entries, including materials, ranges, offsets, and randomization. |
| **Actors** | all `FSPSplineActorConfig` entries. |
| **Mobility** | mobility of the actor, spline, and generated components. |
| **Scale Structure With Actor** | how distances behave when the actor is scaled. |
| **Stable Curve Orientation** | stable orientation for Curve meshes. |
| **Description** | a note inside the asset; the **Save to Preset** command doesn't overwrite it. |

A preset deliberately **doesn't change** the spline's shape and transform, **Loop**, geometry tools, the
temporary reference spline, export folders, or merge settings. These are specific-actor data or personal
editor settings, not part of the style.

## Creating One from a Configured Actor

1. Configure the actor the way the reusable style should look.
2. In **Preset Folder**, choose a Content Browser folder. The default is
   `/Game/SplineCraftFlow/Presets`; it's remembered separately per user and project.
3. Click **Create Preset from Actor**.

The editor creates a unique `P_<actor name>` asset, copies the style into it, assigns the new preset to
the actor, and shows it in the Content Browser. Existing assets aren't overwritten: a number is appended
to a taken name. For multiple selected actors, one preset is created per actor.

Alternative standard route: **Content Browser → right-click → Miscellaneous → Data Asset →
SplineCraft Flow Preset**. Then assign the asset in the **Preset** field.

## Applying and Updating

- **Apply Preset** copies the assigned preset's style onto the actor and immediately rebuilds the
  construction. Just picking an asset in the **Preset** field doesn't apply anything by itself. For
  multiple actors, the command uses each actor's own assigned preset and runs as a single **Undo** step.
- **Save to Preset** overwrites the shared asset with the current settings of one selected actor. Before
  writing, the editor shows the asset's name and asks for confirmation. The preset's description, the
  spline, and **Loop** aren't changed. After the command, the asset is marked as changed — save it with
  the editor's usual command.

If multiple actors reference the same asset, **Save to Preset** updates the asset itself, but other
actors aren't rebuilt automatically. Select them and click **Apply Preset** when you're ready to accept
the new style version.

## Runtime and Blueprint

The preset class and the **Preset** field belong to the Core module and are available at runtime. The
Blueprint function **Apply Preset** takes a `SplineCraft Flow Preset`, copies the style, and rebuilds the
actor; it returns `false` for an invalid asset. Asset creation, Content Browser overwriting, and editor
dialogs stay in the `SplineCraftFlowEditor` module and don't reach the packaged game.

## See Also

- [Static Meshes](03-Static-Meshes.md)
- [Actors Along the Spline](08-Actors-Along-Spline.md)
- [Transform, Mobility, and Blueprint API](09-Transform-Mobility-And-Blueprint.md)
- [Merge to Static Mesh](10-Merge-To-Static-Mesh.md)
