# 08 — Data Asset Editor

*🇬🇧 English | [🇺🇦 Українська](../uk/08-Asset-Editor.md)*

Create a **Fuzzy System** via **Content Browser → Add → Miscellaneous → Data Asset** and open it with a double-click. The editor has four tabs; closed tabs are restored from the **Window** menu.

## Toolbar

- **Save** saves the Unreal package.
- **Load Preset** picks a JSON file from any folder. The document is first parsed and fully compiled. Only after success does the system replace the data of the open Data Asset.
- **Save To JSON** validates the current system and exports a portable preset.

A successful `Load Preset` supports Undo, marks the package as modified, and refreshes all tabs. Cancelling the file dialog or a JSON error doesn't change the Data Asset.

## Details

This is where `Inputs`, `Outputs`, `Rules`, and `Settings` are edited. Each set's header shows a thumbnail of its membership function. Expand a row to change the shape and parameters with standard Unreal controls.

Normal Save, Undo, and Redo work for all authoring changes. Preview values from the Inference tab don't pollute the Data Asset.

## Variables

Select an input or output to see all of its sets on one large graph. The legend links each set to a color. For inputs, a vertical marker shows the preview value, and horizontal markers show the degrees of membership. Hover over the graph to read coordinates.

## Inference

The main area is built for simultaneous observation:

- **Inputs — live controls** on the left: sliders, numeric fields, and degrees of membership;
- **Outputs — live results** on the right: compact cards with a number, an aggregated curve, and a crisp marker;
- **Rule firing strengths — details** at the bottom: a separate scrollable rule panel.

Changing an input immediately updates the entire inference. Outputs stay on screen even when there are many rules. The horizontal and vertical splitters can be dragged.

`Reset inputs` removes temporary overrides and returns inputs to their current `Default Value`. If the range has narrowed, the preview is clamped automatically. A removed input is forgotten.

## Compile Log

Shows compilation diagnostics and warnings from the latest preview. An invalid rule remains visible, has zero activation, and is marked as skipped. Values obtained from a system with errors are a partial preview and should not be used as a valid decision.

Preset format details are in [11 — JSON Presets](11-JSON-Presets.md).
