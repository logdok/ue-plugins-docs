# 01 — Core Concepts

*🇬🇧 English | [🇺🇦 Українська](../uk/01-Core-Concepts.md)*

This chapter explains the data model. Once you've read it you'll understand why
items behave the way they do, and you'll be able to answer the most common
beginner question yourself — "why don't these two items merge?".

---

## An item type and an instance are different things

This is the central split in the plugin.

| | Item type | Instance |
|---|---|---|
| Class | `UISDemoItemDefinition` | `UISDemoItemInstance` |
| What it is | an asset in the Content Browser | a concrete copy in someone's inventory |
| How many | one for the whole game | as many as there are copies in the world |
| What it holds | name, icon, tags, fragments | stack count, durability, charges |
| Example | "Iron Sword" | "that chipped sword in slot 3" |

The item type describes everything that is **the same** for every copy. The
instance holds only what makes a particular copy **different** from another.

That's why a stack of 20 identical potions is **one** instance with
`StackCount = 20`, not twenty objects. This directly affects the stacking rules
(see below).

---

## An item is a set of fragments

The classic alternative is a class hierarchy: `AItem` → `AWeapon` →
`AMeleeWeapon`. It breaks on the first item that doesn't fit the tree: a torch
that is a weapon, a consumable and has a fuel reserve, all at once.

Here an item has **no** behaviour of its own. It only lists **fragments**, and
they do all the work:

| Item | Fragment set |
|---|---|
| Iron Sword | `Equippable` + `Durability` |
| Health Potion | `Stackable` + `Consumable` |
| Copper Ore | `Stackable` + `Weight` |
| Torch | `Equippable` + `Consumable` + `Durability` |
| Quest Letter | *no fragments* — one slot, does nothing |

Giving an item a capability means adding a fragment to the array. Removing one
means deleting an entry. The inventory code never changes.

There are five shipped fragments: `Stackable`, `Equippable`, `Consumable`,
`Durability`, `Weight`. Your own are no more privileged than these. Details in
[04 — Fragments](04-Fragments.md).

> **A rule worth remembering right away.** The fragment object lives inside the
> asset and is **shared by every copy of the item in the game**. Writing
> anything into a fragment field at runtime is always a bug: you'd be changing
> the asset for everyone at once. Mutable state belongs to the instance.

---

## Stacking rules

Two instances merge into one slot only if **all** of the following hold:

1. the same item type;
2. the maximum stack size is greater than 1 (i.e. there is a `Stackable` fragment);
3. **neither of them has its own values** in `StatValues`;
4. every fragment agreed (the `CanStackWith` hook).

Point 3 is the one that surprises people most often, and is also the most
useful.

### Why a chipped sword doesn't merge with a new one

When an item has durability, charges or any other per-instance value, it becomes
**individual**. If the plugin let you merge a sword at 12/100 with a new one,
the player would get either two brand-new swords or two chipped ones — either
way, something vanished without a trace.

So the rule is simple and unconditional: **any per-instance value forbids
stacking**.

The inspector shows this directly: an item with its own values gets a purple
**UNIQUE** badge and a list of those values next to it
([13 — Debugging](13-Debugging.md)).

### Consequence for the Stackable + Durability combination

Such an item will stack only as long as durability hasn't been stamped on it.
Since `Durability` stamps full durability on every new copy, in practice these
items never stack. This is not a bug — it's a direct consequence of the rule.
Just don't add both fragments unless you understand why you want exactly that
behaviour.

---

## Slots are sparse

Slots are numbered `0..MaxSlots-1`, but **internally only the occupied ones
exist**.

A 30-slot inventory with three items stores three entries, and their indices
might be 0, 7 and 29 — there are no empty "holes" in memory.

What this means for your code:

```cpp
// WRONG: entry #N is not slot #N
const FISDemoInventoryEntry& Entry = Inventory->GetAllEntries()[SlotIndex];

// RIGHT: ask about a specific slot
UISDemoItemInstance* Item = Inventory->GetItemAtSlot(SlotIndex);

// RIGHT: iterate what's there, reading the index from the entry itself
for (const FISDemoInventoryEntry& Entry : Inventory->GetAllEntries())
{
    Draw(Entry.SlotIndex, Entry.Instance);
}
```

`MaxSlots = 0` means an **unlimited** inventory. There is no separate flag —
zero is the switch.

---

## Two components

| Component | What it does |
|---|---|
| `UISDemoInventoryComponent` | stores items in numbered slots |
| `UISDemoEquipmentComponent` | wears items in slots named by tags |

They are independent. The inventory works on its own; equipment can also work on
its own (for an NPC whose weapon can't be taken, say). They are linked only when
you say so — and then equipping takes the item out of the backpack, and
unequipping puts it back.

Both can be attached to any actor: a character, an NPC, a chest, a corpse, a
vending machine. A chest is the same inventory component, just on a different
actor.

---

## What makes an item "usable"

The plugin has no built-in notion of "drink", "eat" or "apply". It has
`TryUseItem`, which:

1. asks **every** fragment whether it can be used right now (`CanBeUsed`) — one
   "no" stops everything;
2. then lets **every** fragment apply its effect (`OnUsed`).

The result is an `EISDemoItemUseResult`:

| Value | When |
|---|---|
| `Success` | used |
| `NoItem` | the slot is empty or the index is out of range |
| `NotUsable` | no fragment makes the item usable |
| `Blocked` | a fragment refused (broken tool, no charges, cooldown) |

The last three are **normal, expected answers**, not errors. Show them to the
player as text; don't log them as a failure.

---

## Where to next

- Enable the plugin and find the inventory in your code:
  [02 — Setup](02-Setup.md)
- Create your first item: [03 — Authoring Items](03-Authoring-Items.md)
- Understand how capabilities work: [04 — Fragments](04-Fragments.md)
