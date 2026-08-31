# From asset to a player's hand

*🇬🇧 English | [🇺🇦 Українська](../uk/Story-Walkthrough.md)*

This isn't a reference — it's one short story. The goal: in 10 minutes, see with
your own eyes how an item travels the whole way from a Data Asset to a sword in
a character's hand, without writing a single line of inventory code yourself.
Every example below already lives in the plugin itself
(`Plugins/InventorySystemDemo/Content/Demo/Items/`), so you can open the editor and
repeat each step literally — with no asset of your own.

If after this page you want to understand "but why this way" — the chapters at
the end cover each step in detail.

---

## The cast

Seven items are already created in `Plugins/InventorySystemDemo/Content/Demo/Items/`
— together they show every combination of the shipped fragments (what a fragment
is, and why an item is a set of fragments rather than a class —
[01 — Core Concepts](01-Core-Concepts.md)):

| Asset | What it is | What it's made of |
|---|---|---|
| `DA_IronSword` | Iron Sword | `Equippable` + `Durability` |
| `DA_WoodenShield` | Wooden Shield | `Equippable` (other hand) + `Durability` |
| `DA_KnightHelmet` | Knight's Helmet | `Equippable` (head) + `Durability` — wears not from use |
| `DA_Torch` | Torch | `Equippable` + `Durability` + `Consumable` — all three at once |
| `DA_HealthPotion` | Health Potion | `Stackable` + `Consumable` |
| `DA_CopperOre` | Copper Ore | `Stackable` + `Weight` |
| `DA_AncientRelic` | Ancient Relic | *no fragments at all* |

None of them has a class of its own — neither C++ nor Blueprint. It's just data
on an asset. That's why all the difference in behaviour between the sword, the
shield, the helmet, the torch, the potion, the ore and the relic is visible
below in a few lines per item, not in seven separate classes.

---

## Step 0 — the character needs a backpack

Before you can pick anything up, the player must have an `Inventory` component
(and, if you plan to equip anything — `Equipment`). The fastest way the first
time requires no code and no edit to the character Blueprint:

**Project Settings → Game → Inventory System:**

| Setting | Value for this story |
|---|---|
| `Auto Create Player Inventory` | ✅ |
| `Auto Create Player Equipment` | ✅ |
| `Default Equipment Slots` | `Equipment.Slot.Weapon.Main` |

Done. The plugin creates both components on every player's `PlayerState` as soon
as they appear in the game. For a permanent project the component is usually
added by hand to a specific character — [02 — Setup](02-Setup.md) explains why
auto-creation is off by default and when to turn it on deliberately.

---

## Step 1 — the sword appears in the world

Place an `AISDemoItemPickup` actor in the level and set its `Item Definition` field
to `DA_IronSword`. That's it — no separate Blueprint for "a sword lying on the
ground" is needed: the same `Item Pickup` class serves any item in the game.

The mesh the player sees, the plugin takes from the `PickupMesh` field of
`DA_IronSword` itself (category `Item|World` in Details) — automatically, at game
start. If `PickupMesh` isn't set, the actor still works (it can be picked up),
it just stays invisible on the ground. That's why the **`pickup mesh: yes / no`**
line in the debug panel (`DemoInvOverlay`) is worth checking for a newly created
item before you place it in the world — it says directly whether the player will
see anything on the ground.

---

## Step 2 — the player picks up the sword

Interaction (a key, a trace under the crosshair, a trigger — that's your
project's logic) calls one method — **on the player's inventory, not on the
pickup**:

```cpp
Inventory->TryCollect(Pickup);
```

Why this way, and not `Pickup->TryCollect(...)` directly: the pickup has no
network connection of its own (nobody has "picked it up" yet), so it can't
forward a client's request to the server. The player's inventory has a
connection, so the request goes through it — the same `Try*` as everywhere in
the plugin, called from anywhere and figuring out where it should actually run.
The developer doesn't have to think about it.

The method finds where to put it and places the item in the same state it was in
on the ground (if the sword was picked up damaged rather than freshly placed,
the damage is kept). The pickup is destroyed (the default behaviour), and the
sword now sits in an inventory slot.

**Check it without a single line of code:** open `DemoInvOverlay` (the `I` key or the
console command of the same name), type `iron` into the GIVE ITEM field, press
**Give** — and that same `DA_IronSword` ends up in slot 0. This is exactly the
same action `TryCollect` performs in the game: the panel just lets you click it,
rather than run across the level to test.

---

## Step 3 — the sword ends up in the hand

That's all the magic — and it has exactly zero lines of code on your side. On
`DA_IronSword`, on the `Equippable` fragment, two fields are filled in (category
`Equippable|Visuals`):

| Field | Value in the example |
|---|---|
| `EquippedActorClass` | the sword actor's class (a Blueprint with a static mesh) |
| `AttachSocketName` | `hand_rSocket` (or any socket on your skeleton) |

You call — from a UI button, a hotkey, anywhere:

```cpp
Equipment->TryEquipFromInventorySlot(0);
```

…and the equipment component creates the `EquippedActorClass` itself and attaches
it to the named socket of the character — separately on each machine. This isn't
replicated, so a sword in the hand costs no network traffic beyond the fact that
"this slot is occupied".

If no socket is set, the actor attaches to the character's root rather than
disappearing; on a bare `DefaultPawn` with no skeletal mesh that's what happens,
and for testing the logic that's perfectly fine.

**The same step without code:** in `DemoInvOverlay`, on the sword's row in CONTENTS —
an **Equip** button. Click, and that same `TryEquipFromInventorySlot` fires from
the mouse; the EQUIPMENT panel immediately shows an occupied `Weapon.Main` slot.

> If `EquippedActorClass` is left empty, the item still equips (the slot is
> occupied, `GetTotalStatValue` counts its bonuses) — nothing just appears in
> the hand. Useful for a ring that only grants stats, or for armour already
> baked into the character mesh.

---

## And the shield in the other hand — two equipped items at once

`DA_WoodenShield` is the same recipe as the sword (`Equippable` + `Durability`),
only `EquipmentSlot` points at `Equipment.Slot.Weapon.Offhand` instead of
`Weapon.Main`. The same call, a different slot:

```cpp
Equipment->TryEquipFromInventorySlot(1); // if the shield is in slot 1
```

Now the character holds both items at once — one in each hand, each attached to
its own socket independently. There's no special "weapon-shield pair" handling
in the plugin: `Weapon.Main` and `Weapon.Offhand` are just two different entries
in the equipment component's `AvailableSlots`, and each is handled the same way
as the single slot in the sword example.

**Without code:** the **Equip** button on the shield's row in `DemoInvOverlay` — just
like for the sword.

---

## And the helmet on the head — the same `Durability`, different wear logic

`DA_KnightHelmet` is the third item worn at once (`Equipment.Slot.Armor.Head`, a
third independent slot alongside the two hands), and at first glance the same
recipe: `Equippable` + `Durability`. The difference is one number.

The sword and shield have `DurabilityPerUse` greater than zero: durability is
spent when the item is explicitly *used*. The helmet has `DurabilityPerUse = 0`
— durability isn't spent on its own at all. This isn't an oversight but
documented behaviour: nobody "uses" armour with a click, it wears from the
character being hit. So the project's combat code deducts it directly from the
instance itself, not through the fragment:

```cpp
HelmetInstance->ModifyStatValue(ISDemoTags::Stat_Durability, -DamageTaken);
```

(`Durability->Repair()` won't do here — it only **restores** durability, and a
negative value means "repair fully" to it, not "take away". For external damage
it's `ModifyStatValue` on the instance, with a negative delta.)

…when the character takes a hit — not relying on `TryUseItem`, which makes no
sense for armour at all. `MaxDurability = 60` here is the same kind of number as
on the sword or shield — it counts "how many hits it survives", not "how many
times it was clicked".

**Without code:** the **Equip** button on the helmet's row in `DemoInvOverlay`, like
the rest. There won't be a **Use** button — the helmet doesn't need a
`Consumable` fragment, and without one `IsUsable()` for armour returns false,
just like for copper ore.

---

## And now the torch — all three fragments at once

`DA_Torch` has the same `Equippable` as the sword (it appears in the hand the
same way), plus `Durability` (a "fuel" reserve) and `Consumable` (spent on use).
No separate code was needed for this — just a third entry in the fragment array.

```cpp
Inventory->TryUseItem(SlotIndex, Instigator);
```

The same call any item with a "Use" button uses: the fragments decide what it
means for the torch specifically (spend a charge, reduce the fuel reserve). The
inventory itself knows nothing special about the torch.

**Without code:** the **Use** button on the torch's row in `DemoInvOverlay`.

---

## And the potion — no equipping at all

`DA_HealthPotion` (`Stackable` + `Consumable`) never ends up in EQUIPMENT — it
has no `Equippable` fragment, so the debug panel doesn't even show an **Equip**
button for it (the "does it have the fragment" check hides it automatically, for
any item). Use is the same `TryUseItem`, only this time the stack decreases by
one instead of waiting for wear.

---

## And the copper ore — cargo with no action at all

`DA_CopperOre` (`Stackable` + `Weight`) has no **Use** button at all — it has no
`Consumable`, so `IsUsable()` returns false, and the debug panel hides the
button just as it hid **Equip** for the potion. The only thing this ore does is
take a slot in a stack (up to 99 copies in one slot) and add weight to the
inventory's `GetCurrentWeight()`.

This is a deliberately boring example: not every item has to *do* something. A
crafting material, currency, raw materials — all of these are legitimate items
with empty behaviour, and the plugin doesn't force you to invent a fake "use"
for them.

---

## And the ancient relic — an item with no fragment at all

`DA_AncientRelic` is the same "Quest Letter" from the table in
[01 — Core Concepts](01-Core-Concepts.md), just with a different name. Its
`Fragments` array has zero entries. In practice that means:

- **doesn't stack** — `GetMaxStackSize()` without a `Stackable` fragment always
  returns 1;
- **doesn't equip** — no **Equip** button will appear, as for the ore or the
  potion;
- **does nothing on Use** — there won't be a **Use** button either.

In the debug panel this is shown as plain text: the **Fragments** row for this
item shows `none`, not an empty space. This isn't a broken item — it's the
simplest possible item, and the system makes no exception for it: a quest item,
a key, a story flag — all of these are an `ISItemDefinition` with no fragment,
and nothing more is needed.

---

## What we just saw

None of the seven examples added a single line to the inventory or equipment
code. All the difference between the sword, shield, helmet, torch, potion, ore
and relic is just a different set of fragments (and different values on them —
the helmet and sword have the same fragments but different behaviour via one
number, `DurabilityPerUse`) on the same `ISItemDefinition` class, from three
fragments at once down to zero. That's the plugin's central idea:
[01 — Core Concepts](01-Core-Concepts.md) covers it in detail, with the same
example table.

## Where to next

| You want | Read |
|---|---|
| Create your own item | [03 — Authoring Items](03-Authoring-Items.md) |
| Understand each fragment and its hooks | [04 — Fragments](04-Fragments.md) |
| Build a pickup and equipment UI | [08 — Building the UI](08-Building-UI.md) |
| Place items and chests in a level | [07 — Loot and world items](07-Loot-And-Pickups.md) |
| Everything about `DemoInvOverlay` and console commands | [13 — Debugging](13-Debugging.md) |

---
