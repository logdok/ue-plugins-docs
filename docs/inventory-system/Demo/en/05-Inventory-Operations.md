# 05 — Inventory Operations

*🇬🇧 English | [🇺🇦 Українська](../uk/05-Inventory-Operations.md)*

Everything you can do with an inventory: grant, take, move, use, tidy. Plus
filters, weight and events for the UI.

---

## One rule for every operation

Every method that changes state is named `Try*` and is called **from anywhere**
— client or server, Blueprint or C++:

```cpp
Inventory->TryAddItem(Definition, 3, Remainder);
```

You never write `HasAuthority()` and never choose between a "server" and a
"client" version of the call. Why this works — [09 — Multiplayer](09-Multiplayer.md).

> **What `Try*` returns on a client.** `true` means **"request sent"**, not
> "succeeded" — the final decision is always the server's. So build the UI on
> events, not on the return value. See [events](#events-for-the-ui) below.

---

## Grant an item

```cpp
int32 Remainder = 0;
const bool bAdded = Inventory->TryAddItem(PotionDef, 5, Remainder);

if (Remainder > 0)
{
    // Not everything fit - show "Inventory full" or drop the rest on the ground.
}
```

Existing partial stacks are topped up first, then new slots are opened.
**Partial success is a normal result**, not an error: `Remainder` says how many
didn't fit.

### Find out in advance whether it fits

```cpp
const int32 WillFit = Inventory->CanAcceptItemCount(PotionDef, 5);
```

An honest answer taking into account filters, room in partial stacks, free slots
and the weight limit. This is what to show as a pickup preview or a shop
purchase size.

---

## Take an item

From a specific slot — when the player acted on a slot they can see:

```cpp
Inventory->TryRemoveItemFromSlot(SlotIndex, 1);
```

By type, not caring about slots — when consuming materials:

```cpp
int32 Remainder = 0;
Inventory->TryRemoveItem(IronOreDef, 10, Remainder);
```

The second form spans any number of stacks and empties the **smallest first** —
so leftovers consolidate instead of scattering across the inventory.

> **Check first, then spend.** A partial removal that then "fails" leaves the
> player with no materials and no result:
>
> ```cpp
> if (Inventory->HasItem(IronOreDef, 10))  // all or nothing
> {
>     Inventory->TryRemoveItem(IronOreDef, 10, Remainder);
>     CraftSword();
> }
> ```
>
> Or use `TakeItemFrom` from the library — it does this check itself.

---

## Move between inventories

This is the most important place in the whole chapter, because **the direction
of the call matters**.

```cpp
// Take from a chest (looting)
PlayerInventory->TryTransferFrom(ChestInventory, SlotIndex);

// Put into a chest
PlayerInventory->TryTransferTo(ChestInventory, SlotIndex);
```

Both calls are made **on the player's inventory**, not the chest's. The reason
is purely network: a chest in the world has no network connection, so a client's
request through it wouldn't reach the server
([09 — Multiplayer](09-Multiplayer.md#the-exception-ownerless-containers)).

In Blueprint there's a ready node that picks the direction itself:

```
Move Item Between Actors (From Actor, To Actor, From Slot)
```

### Pick an item up off the floor

The same principle, one more example: `AISDemoItemPickup` (an item lying in the
world — [07 — Loot and world items](07-Loot-And-Pickups.md)) also has no
connection of its own, so pickup is called the same way — on the player's
inventory:

```cpp
PlayerInventory->TryCollect(Pickup);
```

Not `Pickup->TryCollect(...)`. The difference isn't cosmetic: the pickup variant
is a server-only call that does nothing from a client. The inventory variant is
an ordinary `Try*`, like the rest of this page: called from anywhere, and it
decides whether to run the request locally or forward it to the server.

### Quick move

The shift-click behaviour, when the player doesn't pick a destination slot:

```cpp
PlayerInventory->TryQuickMoveTo(ChestInventory, SlotIndex);
```

It merges into matching partial stacks first, then takes the first free slot.
What doesn't fit stays put — a full destination just moves less rather than
destroying items.

---

## Drag-and-drop inside an inventory

```cpp
Inventory->TrySwapSlots(SlotA, SlotB);   // the basis of drag-and-drop
Inventory->TrySplitStack(SlotIndex, 4);  // split 4 copies into a new slot
```

`TrySwapSlots` **merges** stacks if the items are compatible, and works when one
of the slots is empty — so it doubles as a "move here" operation.

A split preserves the state of both halves: half of a chipped stack stays
chipped.

---

## Use an item

```cpp
const EISDemoItemUseResult Result = Inventory->TryUseItem(SlotIndex, GetPawn());
```

What happens is decided by the fragments: a potion is spent, a torch loses
durability, a rock answers `NotUsable`.

| Result | Meaning |
|---|---|
| `Success` | used |
| `NoItem` | the slot is empty |
| `NotUsable` | no fragment makes the item usable |
| `Blocked` | a fragment refused (broken, no charges, cooldown) |

The last three are normal answers. Show them to the player:
`Get Use Result Text (Result)` gives ready text.

`Instigator` (the second argument) is worth passing — it's who the fragments
apply effects to.

---

## Tidying

```cpp
Inventory->CompactStacks();                        // merge partial stacks
Inventory->SortInventory(EISDemoSortMode::ByName);     // sort and compact
```

`CompactStacks` is **gentler**: it merges scattered stacks of the same item
without touching the rest. A backpack with 3+7+12 arrows will have 22 arrows in
the earliest of those slots, and everything else stays where it was.

`SortInventory` rearranges **everything** and packs from slot 0. Modes: `ByName`,
`ByStackCount`, `ByType`, `ByWeight`.

> Sorting visibly moves items under an open UI. Call it from an explicit player
> action (a "Sort" button), not automatically.

---

## Filters: what an inventory accepts at all

| Field | Effect |
|---|---|
| `AllowedItemTags` | if non-empty — the item must carry at least one of these tags |
| `BlockedItemTags` | an item with any of these tags is refused |

`BlockedItemTags` is checked **first** and always wins. This lets you allow a
broad category and carve exceptions out of it:

```
Shop:
  AllowedItemTags: Item.Type.Weapon      ← we trade weapons
  BlockedItemTags: Item.Property.QuestItem  ← but not quest ones
```

Check in advance: `CanAcceptItem(Definition)`.

### Slot restrictions

A single inventory can have specialised areas — with no second component:

```
SlotRestrictions:
  └── [0]: FirstSlot 0, LastSlot 3, RequiredTags: Item.Type.Ammo
```

Slots 0–3 become an ammo belt at the front of an ordinary backpack. Slots not
covered by any rule accept whatever the inventory-wide filters allow.

Check a specific slot: `CanSlotAcceptItem(SlotIndex, Definition)` — this is the
call a drag-and-drop UI should make on hover to show the right cursor.

---

## Weight

| Field | Effect |
|---|---|
| `MaxWeight` | limit; **0 = weight is ignored** |

The unit weight comes from the `Weight` fragment. Items without one weigh
nothing and are never refused by weight.

Slots and weight are **independent** limits: either one can be the binding
constraint.

```cpp
const float Current = Inventory->GetCurrentWeight();  // for an encumbrance bar
```

Weight is calculated and shown even when `MaxWeight = 0` — it just doesn't
constrain. That's why the inspector says `5.50 weight carried (no limit set)`:
the number is honest, there's no limit.

### How to set capacity correctly

**The normal case — in the component's Details panel.** Open the character or
chest Blueprint, select the Inventory component and set `MaxSlots` and
`MaxWeight` there. That value goes into the class defaults, so both the server
and the clients get it the same way, with no replication.

**Changing at runtime is just an assignment, but on the server.**

```cpp
// Backpack upgrade. Do it where you have authority.
if (Inventory->HasContainerAuthority())
{
    Inventory->MaxSlots  = 40;
    Inventory->MaxWeight = 120.f;
}
```

Both fields replicate, so clients see the new capacity and their UI redraws.
Assigning **on a client** does nothing useful: the next update from the server
overwrites it — exactly like any replicated field.

**What doesn't exist:** a global weight default in Project Settings. There's only
`DefaultInventorySlots` for auto-created inventories; their weight is always 0
until you set it yourself.

> **Watch out.** The weight limit doesn't apply to **worn** items: equipping
> takes the item out of the inventory, and its weight disappears from the
> count. If armour should weigh on the player in your game, count that part
> yourself — via `GetTotalStatValue` on the equipment component, for example.

---

## Starting items

`StartingItems` (a map of "item type → count") fills the inventory at the start
of the game, on the server. Handy for a chest's fixed contents, a trader's
stock, or a test character's starting kit.

For random contents use a loot table —
[07 — Loot and world items](07-Loot-And-Pickups.md).

---

## Events for the UI

This is what you build the interface on — **not the `Try*` return value**.

| Event | When it fires |
|---|---|
| `OnInventoryChanged` | any change; carries the change type |
| `OnItemAdded` | a slot that was empty received an item |
| `OnItemRemoved` | a slot emptied (the item is still readable during the call) |
| `OnItemChanged` | the item is the same, but the data changed (stack, durability) |
| `OnAddRejected` | an add was refused; carries the reason text |

All fire on both the server and the clients — the same handler serves both
modes.

```cpp
Inventory->OnInventoryChanged.AddDynamic(this, &UMyWidget::HandleInventoryChanged);
Inventory->OnAddRejected.AddDynamic(this, &UMyWidget::ShowRejectionToast);
```

`OnAddRejected` is worth binding right away: without it the player won't
understand why a pickup did nothing.

### Drawing the grid

```cpp
for (const FISDemoInventoryEntry& Entry : Inventory->GetAllEntries())
{
    DrawSlot(Entry.SlotIndex, Entry.Instance);   // take the index from the entry
}
```

`GetAllEntries` returns **only occupied** slots. The list is sparse — don't
assume entry #N is slot #N
([01 — Core Concepts](01-Core-Concepts.md#slots-are-sparse)).

---

## Where to next

- Equip an item: [06 — Equipment](06-Equipment.md)
- Make a chest with loot: [07 — Loot and world items](07-Loot-And-Pickups.md)
- The full list of methods: [12 — API Reference](12-API-Reference.md)
