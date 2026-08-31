# 02 — Setup

*🇬🇧 English | [🇺🇦 Українська](../uk/02-Setup.md)*

What you need to do to get the plugin working in your project, and how to reach
the inventory from code.

---

## Installation

1. Copy the `Plugins/InventorySystemDemo` folder into your project's `Plugins/`
   folder.
2. **Edit → Plugins → Inventory System** → enable → restart the editor.
3. If your project has a C++ module and you plan to call the plugin from code,
   add the dependencies to your `.Build.cs`:

```csharp
PublicDependencyModuleNames.AddRange(new string[]
{
    "InventorySystemDemo",       // core: items, components, queries
    "InventorySystemDemoWorld",  // only if you need chests and items on the ground
});
```

From Blueprint you don't need to add anything.

**What you do NOT need to do:** subclass `PlayerState`, `GameState`, `GameMode`
or `Character`; configure an input system; add anything to `DefaultEngine.ini`.
The plugin adds no gameplay at all until you create your first item.

### Example items included

The plugin ships seven ready-made `ISItemDefinition` assets, so you have
something to look at before authoring your own:

| Asset | What it shows |
|---|---|
| `DA_IronSword` | `Equippable` + `Durability` — a weapon that wears down |
| `DA_WoodenShield` | the same, for the off hand |
| `DA_KnightHelmet` | `Equippable` + `Durability` with `DurabilityPerUse = 0`: armour damaged not by use but by being hit — your code subtracts durability via `ModifyStatValue` |
| `DA_Torch` | `Equippable` + `Durability` + `Consumable` — three fragments at once |
| `DA_HealthPotion` | `Stackable` + `Consumable` — the classic potion |
| `DA_CopperOre` | `Stackable` + `Weight` — a crafting material with mass |
| `DA_AncientRelic` | an item with no fragments at all: just a thing in a slot |

They live under `InventorySystemDemo Content → Demo → Items`. To see them in the
Content Browser, turn on **Settings → Show Plugin Content**. Pick them apart,
copy them as templates, or just delete them — the plugin doesn't reference
them.

---

## Where the inventory lives

The component can be attached to **any actor**. The plugin has no opinion about
where exactly it should be — but the choice has consequences:

| Where to attach | Consequence |
|---|---|
| **`PlayerState`** | The inventory survives death, respawn and pawn changes. A typical backpack. |
| **`Pawn` / `Character`** | The inventory dies with the body. Extraction shooters, roguelikes, corpse looting. |
| Any actor in the world | A chest, a corpse, a vending machine, a resource node. |

The simplest path is to add the component to the character Blueprint:
**Add Component → Inventory**. Then set `MaxSlots`, filters and the rest in the
Details panel ([05 — Inventory Operations](05-Inventory-Operations.md)).

---

## Automatic component creation

If you'd rather not edit the character Blueprint, the plugin can create the
components itself.

**Project Settings → Game → Inventory System:**

| Setting | Default | What it does |
|---|---|---|
| `Auto Create Player Inventory` | **off** | creates an inventory for every player |
| `Auto Create Player Equipment` | off | and an equipment component as well |
| `Auto Inventory Host` | `PlayerState` | where exactly to attach them |
| `Default Inventory Slots` | 30 | how many slots to give |
| `Default Equipment Slots` | empty | which slots to open |
| `Max Interaction Distance` | **0 (off)** | server-side distance limit for networked requests |
| `Validate Items On Startup` | on | asset check, once per editor session |

> `Max Interaction Distance` is a sanity limit, not an interaction range. It
> only stops a modified client from searching a chest on the other side of the
> map; what a player can actually reach is decided by your own interaction
> system. Set it several times larger than your real range so ordinary latency
> never trips it. More in [09 — Multiplayer](09-Multiplayer.md).

> There is deliberately no weight limit among these settings: an
> auto-created inventory always gets `MaxWeight = 0`, i.e. it ignores weight.
> If you need a limit, set it on the component (in Blueprint or from code). How
> exactly —
> [05 — Inventory Operations](05-Inventory-Operations.md#how-to-set-capacity-correctly).

### Why auto-creation is off by default

Unlike a quest system, where a manager is always needed, an inventory is
usually configured for a specific character. Silently adding a second component
to every pawn would surprise more people than it would please.

Turn it on for prototypes, game jams and projects where "every player has a
backpack" is simply true and editing a Blueprint for it isn't worth the
bother.

### A manually placed component always wins

The subsystem first checks whether the actor already has a component and
**doesn't touch** it. A character with a configured inventory never gets a
second one.

### This works in both single-player and multiplayer

Components are created where the world has authority — that is, in single-player
and on the server. Clients receive them through ordinary replication. One
mechanism covers login, late join, respawn, seamless travel and PIE.

---

## How to find the inventory from code

The worst thing you can do is hard-wire the location:

```cpp
// FRAGILE: breaks the moment the inventory moves to the PlayerState
UISDemoInventoryComponent* Inv = MyCharacter->FindComponentByClass<UISDemoInventoryComponent>();
```

Instead, ask the library:

```cpp
#include "Core/ISDemoInventoryBlueprintLibrary.h"

UISDemoInventoryComponent* Inv = UISDemoInventoryBlueprintLibrary::GetInventoryFor(Actor);
```

`GetInventoryFor` looks in several places in turn: on the actor itself, through
the `IISDemoInventoryInterface` interface, on the pawn's `PlayerState`, on the
controller's pawn. So you can pass it **anything** — `Character`,
`PlayerController`, `PlayerState` — and get the same component.

This matters for a pickup trigger that doesn't know exactly who walked into it.

In Blueprint this is the **Get Inventory For** node. Likewise **Get Equipment
For**.

### Even shorter

For the most common actions you don't even need to find the component:

```
Give Item To      (Actor, Definition, Count)  →  how many actually fit
Take Item From    (Actor, Definition, Count)  →  succeeded or not
Get Item Count For(Actor, Definition)         →  how many it has
Has Item Count    (Actor, Definition, Count)  →  whether it has enough
```

The full list is in [12 — API Reference](12-API-Reference.md).

---

## Tags shipped with the plugin

The plugin ships a starter set of GameplayTags in
`Plugins/InventorySystemDemo/Config/Tags/InventoryTags.ini`:

- `Item.Type.*` — weapon, armour, consumable, materials, currency, ammo…
- `Item.Property.*` — tradeable, droppable, quest, unique…
- `Item.Rarity.*` — from `Common` to `Legendary`
- `Equipment.Slot.*` — hands, head, torso, legs, accessories
- `Item.Stat.*` — durability, charges, quality, level

This is a **starter set, not a requirement**. Delete what you don't need, add
your own (`Item.Type.Vehicle.Hovercraft`, `Equipment.Slot.Implant.Cortex`) —
the plugin doesn't hard-code any of these tags.

The one exception: `Item.Stat.Durability` and `Item.Stat.Charges` are **read**
by the shipped fragments. Renaming them in the `.ini` breaks durability and
charges.

> Tags register automatically — the plugin tells the engine about its own tag
> directory at startup. You don't need to add anything to
> `DefaultGameplayTags.ini`.

---

## Checking that it works

Press Play and open the console (`~`):

```
DemoInvItems          list every item type in the project
DemoInvGive <name> 5  give yourself an item
DemoInvList           show your inventory contents
DemoInvOverlay        open the inspector
```

If `DemoInvList` says "you have no inventory", the component wasn't found. Check
that it's added to the character, or turn on auto-creation.

---

## Where to next

- Create your first item: [03 — Authoring Items](03-Authoring-Items.md)
- Learn to grant and spend items:
  [05 — Inventory Operations](05-Inventory-Operations.md)
- Understand why this works in multiplayer: [09 — Multiplayer](09-Multiplayer.md)
