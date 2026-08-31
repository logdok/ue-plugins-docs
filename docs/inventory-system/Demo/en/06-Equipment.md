# 06 — Equipment

*🇬🇧 English | [🇺🇦 Українська](../uk/06-Equipment.md)*

Wearing items in named slots: a weapon in hand, a helmet on the head, a ring on
a finger.

---

## Slots are tags, not an enum

Which slots exist is set **on the component** in the `AvailableSlots` field. The
plugin never guesses how your character's body is laid out.

```
Medieval RPG:  Weapon.Main, Weapon.Offhand, Armor.Head, Armor.Chest, Accessory.Ring ×2
Mech sim:      Arm.Left, Arm.Right, Torso, Legs, Reactor
Card game:     (no component needed at all)
```

An item declares where it asks to go, through its `Equippable` fragment. It can
only be equipped where such a slot **actually exists**.

> This is the most common reason for "why won't it equip": the item asks for a
> slot that isn't in this character's `AvailableSlots`. The inspector shows both
> sides — [13 — Debugging](13-Debugging.md).

---

## Component setup

**Add Component → Equipment.**

| Field | Description |
|---|---|
| `AvailableSlots` | the body plan: which slots exist at all |
| `LinkedInventory` | the backpack items are taken from and returned to |
| `bAutoLinkInventoryOnSameActor` | find the inventory on the same actor (**on** by default) |

You can leave `LinkedInventory` alone: the component finds the inventory on its
own actor. The search is **lazy** — it works even if the inventory was added
later.

**Leaving the link empty** is appropriate when equipment lives on its own: an
NPC's weapon that can't be taken, or a mannequin for a preview.

---

## Equipping and unequipping

The most convenient form — the player picks an **item**, not a place:

```cpp
Equipment->TryEquipFromInventorySlot(InventorySlotIndex);
```

The item knows which slot it asks for. This is the call behind an "Equip"
context-menu entry.

The explicit form, when the slot is known:

```cpp
Equipment->TryEquipItem(Item, MainHandTag, /*bRemoveFromInventory*/ true);
```

Unequipping:

```cpp
Equipment->TryUnequipSlot(MainHandTag, /*bReturnToInventory*/ true);
Equipment->TryUnequipAll(true);
```

### Weapon swap — one call

If the slot is occupied, the previous item is **unequipped automatically** and
returned to the backpack. You don't need to unequip separately.

### A full backpack doesn't destroy the item

If there's no room in the inventory, the unequip **doesn't happen**: the item
stays equipped and the call returns `false`. Nothing is lost to an overfull
backpack.

This also covers the case where there are no free slots but there's room
**inside a stack**: the item merges there. And if a fragment gate refuses at the
last moment, the item is returned to itself, and the call honestly returns
`false`.

### Swapping between slots

```cpp
Equipment->TrySwapSlots(MainHandTag, OffHandTag);
```

Works **if both items agree to live in the other's slot**. The agreement is
expressed in the `Equippable` fragment: the slot must be named either as
`EquipmentSlot` or in `AlternativeSlots`.

```
DA_Sword
└── Equippable
    ├── EquipmentSlot   : Equipment.Slot.Weapon.Main    ← where it goes by default
    └── AlternativeSlots: Equipment.Slot.Weapon.Offhand ← and where else it can be
```

A sword that only knows the main hand won't move to the off hand — and the call
returns `false`. This isn't an implementation limitation; it's the same rule
that stops you equipping a helmet on a leg. Items that genuinely are worn in two
places just need to say so.

One of the slots can be empty — then it's simply "move to the other hand".

---

## The visual actor

If the `Equippable` fragment names an `EquippedActorClass`, the actor is created
and attached **on every machine** — on the server and on the clients.

It is **not replicated**: each client creates its own. So a sword in a
character's hand costs exactly as much traffic as the slot entry itself.

Where to attach is decided by `GetAttachmentComponent`. By default this is the
owner's first skeletal mesh, or the root if there isn't one. Override it in
Blueprint if your character holds sockets elsewhere: a turret on a vehicle, a
secondary mesh.

Tweaks after attachment go in `OnEquipmentSpawned` (material by rarity, scale,
pass the actor a reference to the instance) or the `OnEquipmentActorSpawned`
event.

---

## Aggregate stats

The number a character sheet shows as "total armour":

```cpp
const float TotalArmor = Equipment->GetTotalStatValue(MyTags::Stat_Armor);
```

It sums the stat **across everything worn**. It reads values from the instances,
so an upgraded or damaged item contributes its **actual** value, not the
nominal one for the type.

Where the values come from:

1. `Equippable.GrantedStats` stamps them on a new copy at creation;
2. then your game can change them on a particular copy
   (`Instance->SetStatValue(...)`) — sharpening, enchantment, wear.

The plugin applies nothing to the character itself: no two games model armour
the same way. It only gives you a number.

---

## Using a worn item

```cpp
Equipment->TryUseEquippedItem(MainHandTag, GetPawn());
```

Runs the same fragments as `TryUseItem` for the inventory: a broken tool refuses
the same way — in the hand or in the backpack.

---

## Queries for the UI

| Method | What it gives |
|---|---|
| `GetEquippedItem(Slot)` | the item in the slot, or `null` |
| `GetOccupiedSlots()` | which slots are occupied |
| `GetAllEntries()` | everything worn in one pass |
| `HasSlot(Slot)` | whether the character has this slot at all |
| `CanEquipToSlot(Item, Slot)` | whether it can be equipped here — for highlighting a target |
| `GetDesiredSlotForItem(Item)` | where the item asks to go |
| `GetEquippedActor(Slot)` | the slot's visual actor |

For a "paper doll" panel, iterate **`AvailableSlots`**, not the occupied slots —
an empty slot needs drawing too:

```cpp
for (const FGameplayTag& Slot : Equipment->AvailableSlots)
{
    DrawSlot(Slot, Equipment->GetEquippedItem(Slot));  // may be null
}
```

---

## Events

| Event | When |
|---|---|
| `OnItemEquipped` | a slot **became** occupied — the equip itself |
| `OnItemUnequipped` | a slot freed up (the item is still readable) |
| `OnEquippedItemChanged` | the item is the same, its data changed: durability, charges, stats |
| `OnEquipmentActorSpawned` | the visual actor was created and attached |

The first three fire on both the server and the clients.

> **Why a change is a separate event.** A sword you strike with loses one point
> of durability per hit, and every such loss flags the slot for replication. If
> this counted as "equipping", the equip sound would play on every hit. So: the
> durability bar on the character panel — `OnEquippedItemChanged`; the sound,
> the flash and the stat recompute — `OnItemEquipped`.
>
> This is exactly the same trio as the inventory's (`OnItemAdded` /
> `OnItemRemoved` / `OnItemChanged`), just with the slot addressed by a tag
> rather than a number.

---

## Where to next

- Create an item that can be equipped: [03 — Authoring Items](03-Authoring-Items.md)
- Get to grips with the `Equippable` fragment: [04 — Fragments](04-Fragments.md#equippable--the-item-can-be-worn)
- Save what's worn: [10 — Serialization](10-Serialization.md)
