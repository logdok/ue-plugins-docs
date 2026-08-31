# InventorySystemDemo User Guide

*🇬🇧 English | [🇺🇦 Українська](../uk/README.md)*

A data-driven inventory and equipment system for **Unreal Engine 5.7** with
full multiplayer support.

Items are authored as Data Assets, gameplay code calls a single method, and the
plugin does the rest — including network replication.

> **Version 1.0** · Runtime · UE 5.7 · [Release Notes](Release-Notes.md)

---

## Quick start

**1. Enable the plugin.** Copy `Plugins/InventorySystemDemo` into your project's
`Plugins/` folder and enable it: **Edit → Plugins → Inventory System**.

**2. Create an item.** Content Browser → right-click →
**Miscellaneous → Data Asset → ISItemDefinition**. Name it `DA_HealthPotion`, set
`DisplayName`, and in the **Fragments** field add:

- **Stackable** → `MaxStackSize = 10`
- **Consumable** → `ConsumeAmount = 1`

> Don't want to author anything just to look around? The plugin ships seven
> ready-made items under `InventorySystemDemo Content → Demo → Items` (turn on
> **Show Plugin Content** in the Content Browser settings to see them). A sword,
> shield, helmet, torch, potion, ore and relic — together they cover all five
> fragments.

**3. Give a character an inventory.** Character Blueprint →
**Add Component → Inventory**.

**4. Try it.** Press Play and open the console:

```
DemoInvGive Potion 5      give yourself 5 potions
DemoInvOverlay            open the inspector
```

Done — items are granted, stacked and used.

The same from code:

```cpp
int32 Remainder = 0;
InventoryComponent->TryAddItem(PotionDefinition, 5, Remainder);
```

That line works **both in single-player and on a multiplayer client** — with no
`HasAuthority()` branch on your side.

---

## The three ideas the plugin rests on

If you read only one paragraph, read this one.

**1. An item is a set of fragments, not a class.**
There is no `Weapon` or `Potion` base class. A sword is `Equippable + Durability`,
a potion is `Stackable + Consumable`, a torch is all three at once:
`Equippable + Durability + Consumable` (held in hand, has a fuel reserve, spent
on use). "All three" means three of the four types just mentioned, not every
fragment in the plugin: there are five in total (`Stackable`, `Equippable`,
`Consumable`, `Durability`, `Weight`). Adding a capability = adding a fragment;
the inventory code doesn't change.

**2. One API for any game mode.**
Every method that changes state is named `Try*` and is callable from anywhere.
In single-player it runs in place and creates no RPC at all; on a client it
forwards itself to the server.

**3. What makes a copy unique lives in the instance.**
A sword at 12/100 durability and a fresh sword reference the same asset but hold
different per-instance values — which is exactly why they don't merge into a
stack.

---

## Who should read what

| You | Start with |
|---|---|
| **Total beginner** — want to see a result in 10 min, not read chapter by chapter | [From asset to a player's hand](Story-Walkthrough.md) |
| **Designer** authoring items | [01](01-Core-Concepts.md) → [03](03-Authoring-Items.md) → [04](04-Fragments.md) |
| **Gameplay programmer** | [05](05-Inventory-Operations.md) → [11](11-Recipes.md) |
| **Reading the code / extending the plugin** | [Architecture](Architecture.md) → [04](04-Fragments.md) |
| **UI programmer** | [08](08-Building-UI.md) → [12](12-API-Reference.md) |
| **Multiplayer programmer** | [01](01-Core-Concepts.md) → [09](09-Multiplayer.md) |
| **QA / tooling** | [13](13-Debugging.md) |

Something behaving oddly? [14 — FAQ](14-FAQ.md) has a "why isn't it working"
checklist.

---

## Chapters

| # | Chapter | What's inside |
|---|---|---|
| – | [From asset to a player's hand](Story-Walkthrough.md) | A 10-minute story: a sword, a torch and a potion from Data Asset to the player's hand |
| – | [Release Notes](Release-Notes.md) | What's here, what changed, what's not there yet |
| 01 | [Core Concepts](01-Core-Concepts.md) | Type and instance, fragments, stacking rules, sparse slots |
| 02 | [Setup](02-Setup.md) | Installation, Project Settings, where the inventory lives and how to find it |
| 03 | [Authoring Items](03-Authoring-Items.md) | Field reference and ready recipes: potion, sword, ore, ammo |
| 04 | [Fragments](04-Fragments.md) | The five shipped fragments, every hook, how to write your own |
| 05 | [Inventory Operations](05-Inventory-Operations.md) | Add, move, use, sort; filters, weight, events |
| 06 | [Equipment](06-Equipment.md) | Slots as tags, equipping, visual actors, aggregate stats |
| 07 | [Loot and world items](07-Loot-And-Pickups.md) | Loot tables, chests, items on the ground, a custom container |
| 08 | [Building the UI](08-Building-UI.md) | Grid, icons, tooltips, drag-and-drop, a chest window |
| 09 | [Multiplayer](09-Multiplayer.md) | Why the same code works in both modes; the one exception |
| 10 | [Serialization](10-Serialization.md) | `ExportState` / `ImportState`, embedding into your save game |
| 11 | [Recipes](11-Recipes.md) | Crafting, trading, a hotbar, looting a corpse, dropping, repair |
| 12 | [API Reference](12-API-Reference.md) | The full public API |
| 13 | [Debugging](13-Debugging.md) | Inspector, console commands, logs, a problem checklist |
| 14 | [FAQ](14-FAQ.md) | Short answers to common questions |
| – | [Architecture](Architecture.md) | How the plugin is built internally, and why |

---

## Conventions

- **"Item type"** is a `UISDemoItemDefinition` asset. **"Instance"** is a concrete
  copy in someone's inventory (`UISDemoItemInstance`).
- Code snippets are C++ unless noted otherwise. **Nearly every function shown is
  also callable from Blueprint**; the two exceptions are marked *(C++ only)* in
  [12 — API Reference](12-API-Reference.md) and add no capability.
- "Inventory" means `UISDemoInventoryComponent`, "equipment" means
  `UISDemoEquipmentComponent`.
