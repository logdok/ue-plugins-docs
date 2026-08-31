# 14 — FAQ

*🇬🇧 English | [🇺🇦 Українська](../uk/14-FAQ.md)*

Short answers, with links to the chapter where each is covered in full.

- [First steps](#first-steps)
- [Items and stacking](#items-and-stacking)
- [Slots and capacity](#slots-and-capacity)
- [Equipment](#equipment)
- [The UI](#the-ui)
- [Multiplayer](#multiplayer)
- [Serialization](#serialization)
- [Extending](#extending)

---

## First steps

### Do I need to write C++?

No. Items are Data Assets, components attach to any actor, every plugin function
is callable from Blueprint. Even a custom fragment can be a Blueprint class
derived from `ISItemFragment`.

[11 — Recipes](11-Recipes.md) exists in case you *want* C++, not because it's
required.

### Is there a ready example I can just run?

Yes — `Content/Demo/Maps/Demo.umap` (it's the project's startup map). Seven
pickup stations, one per fragment combination, two chests with different loot
roll modes. Controls: `WASD` + mouse, `E` — pick up an item or open a chest,
`F` — hit with what's in the main hand, `I` — the inventory inspector.

The text version of the same story is
[From asset to a player's hand](Story-Walkthrough.md).

### Do I need to subclass `Character`, `PlayerState` or `GameMode`?

No. Add a component to any actor and it all works. If you don't want to edit the
character Blueprint, turn on auto-creation in
[Project Settings](02-Setup.md#automatic-component-creation).

### Where should item assets live?

Anywhere in your content. Discovery goes through the Asset Registry by class, so
the folder structure is yours, and moving assets breaks nothing.

The one caveat — **don't rename an asset after release**: its name is written
into players' saves. `DisplayName` can be changed at any time.

### Can I add the plugin to a project already in development?

Yes. It doesn't require a particular base class, input system or project
structure, and adds no gameplay until you create your first item.

### Does all of this work in single-player?

Yes, with no overhead at all. In single-player calls run in place and create no
RPC at all — [09 — Multiplayer](09-Multiplayer.md).

---

## Items and stacking

### Why don't two identical items merge?

The most common reason: **one of them has its own values** (durability,
charges). Any per-instance value makes a copy individual — otherwise a chipped
sword, merged with a new one, would destroy the difference.

Open the inspector (`DemoInvOverlay`): an item with its own values gets a purple
**UNIQUE** badge.

The second reason — there's no `Stackable` fragment, so the stack max is 1.

Details in [01 — Core Concepts](01-Core-Concepts.md#stacking-rules).

### How do I make an item that doesn't stack?

Don't add the `Stackable` fragment. That's the default state.

### Can I combine `Stackable` and `Durability`?

Technically yes, in practice the item will never stack, because `Durability`
stamps durability on every new copy, which makes it unique. If you need exactly
that combination, you probably want something else.

### How do I make an item with several uses that doesn't disappear?

`Consumable` with `MaxCharges > 0`. The item stays in the slot and spends
charges: a wand of 10 spells, a lantern with fuel, a medkit with 3 uses.
[04 — Fragments](04-Fragments.md#consumable--the-item-is-spent).

### I need a capability that isn't among the shipped fragments.

Write your own fragment — in Blueprint or C++. Your fragment is no more limited
than the built-in ones: they're themselves ordinary subclasses of
`ISItemFragment`. [04 — Fragments](04-Fragments.md#a-custom-fragment).

### How do I attach my own data to an item?

Numeric values — in the instance's `StatValues` (`SetStatValue`). Data shared
across the type — fields of your own fragment.

---

## Slots and capacity

### How do I make an unlimited inventory?

`MaxSlots = 0`. There's no separate flag — zero is the switch.

### How do I restrict an inventory to certain item types?

`AllowedItemTags` — accept only what's listed; `BlockedItemTags` — refuse what's
listed. The second is checked first and always wins.
[05 — Operations](05-Inventory-Operations.md#filters-what-an-inventory-accepts-at-all).

### How do I make dedicated ammo slots?

`SlotRestrictions` on the component: a slot range + required tags. A single
inventory can have an ammo belt at the front and ordinary slots after — no
second component needed.

### How do I handle an item not fitting?

`TryAddItem` returns `OutRemainder` — how many didn't fit. This is a normal
result, not an error: spill the rest on the ground or show a message.

To find out in advance: `CanAcceptItemCount(Def, Count)` — an honest answer
taking into account filters, stacks, slots and weight.

### Can I have several inventories on one character?

Yes — just add several components. But `GetInventoryFor` returns the **first one
found**; if you need a specific one, keep the reference explicitly or implement
`IISDemoInventoryInterface`.
[11 — Recipes](11-Recipes.md#several-inventories-on-one-actor).

### How do I turn on weight?

Set `MaxWeight > 0` on the component and give items the `Weight` fragment. Items
without that fragment weigh nothing.

Weight and slots are independent limits: either can be the binding constraint.

### Will a client see a `MaxSlots`/`MaxWeight` change during play?

Yes, both fields replicate. A value set in Blueprint reaches clients on its own
— replication matters specifically for a **runtime change**: a backpack upgrade,
or auto-creation with a non-default value in Project Settings.
[05 — Operations](05-Inventory-Operations.md#how-to-set-capacity-correctly).

---

## Equipment

### Why won't an item equip?

Three reasons by frequency:

1. The item has no `Equippable` fragment.
2. The slot the item asks for is **missing** from the character's
   `AvailableSlots`.
3. The slot in the fragment and the slot in the component are different tags (a
   typo).

`DemoEquipList` shows which slots the character has; the inspector — which fragments
the item has.

### How do I make two ring slots?

Give both the same tag (`Equipment.Slot.Accessory.Ring`) and add it to
`AvailableSlots` twice… — no, a tag in the container is unique. Make two
different tags (`...Ring.Left`, `...Ring.Right`) and two different ring types,
or make the slots interchangeable in your own UI.

### Why won't an item unequip?

Most likely, **there's no room in the inventory**. The plugin deliberately
refuses rather than destroy the item: it stays equipped and the call returns
`false`.

### How do I compute total armour?

`Equipment->GetTotalStatValue(YourArmorTag)`. It sums the stat across everything
worn, reading the instances' actual values.
[06 — Equipment](06-Equipment.md#aggregate-stats).

### Is the weapon's visual actor replicated?

No — each machine creates its own from the `Equippable` fragment. So a sword in
the hand costs no traffic beyond the slot entry.

### Why does `OnItemEquipped` fire again when a worn item's durability just changed?

It shouldn't. `OnItemEquipped` means exactly "just equipped"; for a change to
the data of an already-worn item (durability, charges) there's a separate event,
`OnEquippedItemChanged`. If the equip sound plays on every weapon hit — check
which event you subscribed to. [06 — Equipment](06-Equipment.md#events).

---

## The UI

### Can I manipulate the inventory from the inspector without writing code, for a quick check?

Yes. `DemoInvOverlay` isn't just a view: the `Give`/`Fill`/`Use`/`Equip`/`Unequip`
buttons right in the panel call the same `Try*` methods your gameplay code does,
so it's an honest test of behaviour, not a separate toy copy.

### The console commands (`DemoInvGive`, `DemoInvList`…) don't work.

Type `EnableCheats` first — the engine creates a `CheatManager` only in
standalone or on the server, and without this every command answers "Command not
recognized". The `I` key doesn't have this limitation.
[13 — Debugging](13-Debugging.md#console-commands).

### The `I` key doesn't open anything.

Check the **GameMode Override** in this map's World Settings. It silently swaps
the controller for its own, and the key binding lives in
`AHostInventorySystemPlayerController` (or your equivalent) — with a different
controller it simply doesn't exist.

### How do I draw a slot grid?

Iterate the range `0..MaxSlots-1` and ask `GetItemAtSlot(Slot)` — that way you
see the empty cells too. **Don't** index `GetAllEntries()`: that list is sparse,
entry #N is not slot #N. [08 — Building the UI](08-Building-UI.md#drawing-the-grid).

### The icon doesn't show.

`Icon` is a soft reference; it must be loaded: `Async Load Asset` in Blueprint or
`StreamableManager` in C++. This is deliberate, so a thousand item types don't
drag a thousand textures into memory.

### The UI doesn't update.

Subscribe to `OnInventoryChanged` (or the narrower events) instead of polling
every frame. Check that the subscription happened **after** the component
appeared — on a client it can arrive later than the widget.

### The player picks up an item — nothing happens.

Bind `OnAddRejected`: it carries ready reason text ("Inventory full", "Too
heavy"). Without it the refusal looks like a silent bug.

### How do I show durability in a tooltip?

Ask the item for the fragment and take the percentage:

```cpp
if (const UISDemoFragment_Durability* Dur = Def->FindFragment<UISDemoFragment_Durability>())
{
    Bar->SetPercent(Dur->GetDurabilityPercent(Instance));
}
```

---

## Multiplayer

### What does `Try*` return on a client?

`true` means **"request sent"**, not "succeeded". The server decides finally.
Build the UI on events, not on the return value.

### Why doesn't taking from a chest work on a client?

You're most likely calling a method on the **chest's** inventory. It has no
network connection, so the request doesn't reach the server.

Call it on the **player's** inventory:

```cpp
PlayerInventory->TryTransferFrom(ChestInventory, SlotIndex);
```

This is the only place where the network leaks into the API —
[09 — Multiplayer](09-Multiplayer.md#the-exception-ownerless-containers).

### Picking up an item doesn't work on a client.

You're most likely calling `Pickup->TryCollect(...)` directly. The pickup has no
network connection, so the request goes nowhere. Call it on the **player's**
inventory:

```cpp
PlayerInventory->TryCollect(Pickup);
```

The same principle as for chests —
[07 — Loot and world items](07-Loot-And-Pickups.md#pickup).

### `Container->Open()` from a client does nothing.

Unlike the rest of the API, `Open`/`Close` on `AISDemoLootContainer` **don't
forward themselves** — the chest has no connection through which to reach the
server. Call them from the server side of your interaction (a Server RPC on the
player's interaction component; taking items **after** opening no longer needs
this care — `TryTransferFrom` routes itself).
[07 — Loot and world items](07-Loot-And-Pickups.md#it-deliberately-has-no-mesh-or-collision).

### Do other players see my inventory?

Yes, the contents replicate to all observers of the owning actor. This follows
from the same component serving both a player's backpack and a chest, whose
contents everyone should see.

If you need strict privacy — hide the data in your own UI.

### Where's better to keep the inventory — on `PlayerState` or on `Pawn`?

`PlayerState` — the inventory survives death and respawn (a typical backpack).
`Pawn` — the inventory dies with the body (extraction shooters, roguelikes,
corpse looting).

### Do I need to change anything for single-player?

No. The same code and the same map work in both modes with no branches.

---

## Serialization

### How do I save the inventory?

`ExportState()` hands you a plain struct — put it into your own `USaveGame`.
Back — `ImportState()`. [10 — Serialization](10-Serialization.md).

### Is durability saved?

Yes. Every instance value is saved, and on load the creation hooks are **not**
run — a rusty sword loads rusty.

### What happens if an item was removed from the game in a patch?

It's skipped with a warning, the rest loads normally. One removed item doesn't
cost the player the whole save.

### Why does `ImportState` do nothing?

It's a server method. On a client it's ignored with a warning in the log.

### Is equipment saved separately from the inventory?

Yes — `UISDemoEquipmentComponent` has its own `ExportState()`/`ImportState()`,
independent of the inventory's. Put both structs into your `USaveGame`.

---

## Extending

### How do I change inventory behaviour globally?

Subclass `UISDemoInventoryComponent` and name your class in
**Project Settings → Game → Inventory System → Inventory Component Class**.

### Why doesn't my fragment keep values between uses?

Because you're writing to a fragment field, and it's **one for the whole game**
— shared by every copy of the item. Mutable state belongs to the instance:

```cpp
Instance->SetStatValue(MyTag, Value);   // this way
MyField = Value;                        // no - changes the asset for everyone
```

This is the main fragment rule —
[04 — Fragments](04-Fragments.md#the-main-rule-fragments-are-shared).

### How do I make my own container without subclassing `AISDemoLootContainer`?

Add a `UISDemoInventoryComponent` to your actor — that's the container. The ready
class only adds a loot table and an "open/emptied" state.
[07 — Loot and world items](07-Loot-And-Pickups.md#your-own-container-without-a-ready-class).

### How do I grant a player something from an arbitrary place in the code?

```cpp
UISDemoInventoryBlueprintLibrary::GiveItemTo(Player, ItemDef, Count);
```

It finds the inventory itself and returns how many actually fit.

### I set tags in my class's constructor, and a slot or filter turns out empty.

The classic initialisation-time trap: a native class's CDO constructor can run
**before** `FDemoInventorySystemModule::StartupModule` registers the plugin's tag
directory, so `RequestGameplayTag` silently returns an empty tag that time.
Move the tag request into `BeginPlay` — by then the engine is fully started. The
same story with soft references to the item assets themselves
(`TSoftObjectPtr`): construct the references in the constructor all you like, but
call `LoadSynchronous()` in `BeginPlay`.

### I added my tag straight into the `.ini` file, and it doesn't appear.

The format isn't validated in any way and so fails silently: the section must be
`[/Script/GameplayTags.GameplayTagsList]` (not `GameplayTagsSettings`), and
entries must be `GameplayTagList=(Tag="...")` **without** a leading `+`. Both
mistakes produce an empty tag list with no message anywhere. The most reliable
way is to edit tags through Project Settings → GameplayTags in the editor: it
writes the correct format itself.
[02 — Setup](02-Setup.md#tags-shipped-with-the-plugin).

### I want to make a collectible object that isn't an `AISDemoItemPickup`.

Implement `IISDemoCollectibleInterface` (one method — `TryGiveContents`) on your
actor. `UISDemoInventoryComponent::TryCollect` will see it just like an ordinary
pickup. [12 — API Reference](12-API-Reference.md#interfaces--integration-points).
