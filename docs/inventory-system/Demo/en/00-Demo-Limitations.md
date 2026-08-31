# Demo Limitations

*🇬🇧 English | [🇺🇦 Українська](../uk/00-Demo-Limitations.md)*

This is the **evaluation edition** of InventorySystem. It installs alongside the
full plugin in the same project and exists so you can try every feature before
buying — it is not a cut-down replacement for it.

---

## What works exactly like the full plugin

Every feature and every mode works exactly as in the full edition. Nothing is
stubbed out or feature-limited:

- all five shipped fragments (`Stackable`, `Equippable`, `Consumable`,
  `Durability`, `Weight`) and custom fragments of your own;
- stacking, slot restrictions, weight limits, filters;
- equipment slots, using an equipped item, equipment actors;
- loot tables, world containers, dropped-item pickups;
- save / load (`ExportState` / `ImportState`);
- the full network-authority model — the same `Try*` call works on a client
  with no networking code on your side;
- zero-config auto-attach via Project Settings;
- the on-screen inspector and the `DemoInv*` / `DemoEquip*` console commands.

---

## What's restricted

- **Distinct item types per play session — 5.** The demo accepts up to five
  *different* item definitions across every inventory and equipment component in
  the session. Adding more of a type it has already seen is free; the sixth
  brand-new type is refused (`TryAddItem` and friends return `false`), with a
  line in the log and on screen explaining why. The count resets every time a
  game world begins play (press Play again, or restart the session).
- **On-screen watermark.** A small yellow overlay line shows this is the demo
  edition and how much of the session budget is used
  (`DEMO EDITION - 3/5 item types used this session`).
- **Shipping builds.** The plugin refuses to activate in a `Shipping`
  configuration — it is for evaluation in the Editor and Development builds
  only. A Shipping build can still be produced normally; the plugin simply does
  nothing in it (no world subsystem, no auto-attach, no watermark, no budget
  bookkeeping).

Nothing else is limited. If a capability of the full plugin is missing here,
that's a bug — please report it.

---

## Both editions installed at once

If the full InventorySystem plugin is also installed in the same project, this
demo edition detects it and **stands down automatically** — it creates no
components and draws no watermark, staying out of the full plugin's way. To run
both side by side anyway, uncheck **Stand Down When Full Plugin Present** under
**Project Settings → Game → Inventory System (Demo)**.

---

## Upgrading to the full edition

The demo's classes and data assets are a **separate type** from the full
edition's — `UISDemoItemDefinition` with primary asset type `DemoItemDefinition`,
not `UISItemDefinition` / `ItemDefinition`. Item definitions you author against
the demo while evaluating **do not carry over** to the full plugin
automatically; re-create them (or re-parent them) against `UISItemDefinition`
once you switch. Everything you learned about authoring them transfers exactly —
only the asset class changes.
