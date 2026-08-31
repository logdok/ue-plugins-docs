# 12 — API Reference

*🇬🇧 English | [🇺🇦 Українська](../uk/12-API-Reference.md)*

The full public API. Expanded explanations with examples are in the tooltips
right in the editor: hover over any field or node.

**Nearly everything listed here is callable from both Blueprint and C++.** The
exceptions are marked *(C++ only)* and there are just two: `GetEntriesView` on
both containers, which returns an array without a copy, and the
`IISDemoInventoryInterface` interface. They add no **capability** to the plugin —
both have Blueprint equivalents (`GetAllEntries` and component lookup via
`UISDemoInventoryBlueprintLibrary`).

---

## `UISDemoInventoryBlueprintLibrary` — shortcuts

The shortest path to common actions. These functions find the component
themselves, so you don't need to look it up by hand.

| Function | Description |
|---|---|
| `GetInventoryFor(Actor)` | find the inventory — on the actor, its `PlayerState` or pawn |
| `GetEquipmentFor(Actor)` | the same for equipment |
| `HasInventory(Actor)` | whether there's an inventory at all |
| `GiveItemTo(Actor, Def, Count)` | grant; returns **how many actually fit** |
| `TakeItemFrom(Actor, Def, Count, bAllowPartial)` | take; "all or nothing" by default |
| `GetItemCountFor(Actor, Def)` | how many it has |
| `HasItemCount(Actor, Def, Count)` | whether it has enough |
| `MoveItemBetweenActors(From, To, Slot)` | move **in the right direction** |
| `GetItemsWithTag(Actor, Tag)` | every item with a tag |
| `GetCarriedWeight(Actor)` | total weight |
| `GetItemDisplayText(Instance)` | "Sword" or "Arrow x12" |
| `GetUseResultText(Result)` | text for the use result |

---

## `UISDemoInventoryComponent`

### Configuration

| Field | Description |
|---|---|
| `MaxSlots` | number of slots; **0 = unlimited**. Replicated |
| `MaxWeight` | weight limit; **0 = weight is ignored**. Replicated |
| `AllowedItemTags` | if set — only items with one of these tags are accepted |
| `BlockedItemTags` | items with these tags are refused (checked **first**) |
| `SlotRestrictions` | rules for slot ranges |
| `StartingItems` | items granted at startup (on the server) |

### Actions — callable from anywhere

| Method | Description |
|---|---|
| `TryAddItem(Def, Count, OutRemainder)` | add copies; `OutRemainder` is how many didn't fit |
| `TryAddItemInstance(Instance, PreferredSlot)` | add an existing object preserving its state *(server)* |
| `TryRemoveItemFromSlot(Slot, Count)` | remove from a specific slot |
| `TryRemoveItem(Def, Count, OutRemainder)` | remove by type, spanning several stacks |
| `TryTransferTo(Target, FromSlot, ToSlot, Count)` | give to another inventory |
| `TryTransferFrom(Source, FromSlot, ToSlot, Count)` | take from another inventory |
| `TryCollect(Collectible)` | **pick an item up from the world** — call on the player's inventory, not the pickup |
| `TryQuickMoveTo(Target, FromSlot)` | "shift-click": the target picks the spot |
| `TrySplitStack(Slot, SplitCount)` | split a stack |
| `TrySwapSlots(A, B)` | swap places or merge |
| `TryMoveToSlot(From, To)` | the same, in other words |
| `TryUseItem(Slot, Instigator)` | use; returns `EISDemoItemUseResult` |
| `TryClearInventory()` | empty it |
| `SortInventory(Mode)` | sort and compact from slot 0 |
| `CompactStacks()` | merge partial stacks without rearranging the rest |

### Queries — safe on a client

| Method | Description |
|---|---|
| `GetItemAtSlot(Slot)` | the item in the slot, or `null` |
| `GetItemCount(Def)` | how many copies in total |
| `HasItem(Def, Count)` | whether there are at least that many |
| `GetAllItems()` | every item |
| `GetAllEntries()` | every **occupied** slot with its index |
| `GetEntriesView()` | the same, without an array copy *(C++ only)* |
| `FindItemSlot(Def)` | the first slot with this type |
| `FindAllItemSlots(Def)` | every slot with this type |
| `FindSlotOfInstance(Instance)` | where exactly **this** object is |
| `GetTotalItemCount()` | the sum of all stacks |
| `GetOccupiedSlotCount()` / `GetFreeSlotCount()` | occupied / free |
| `GetCurrentWeight()` | current weight |
| `IsFull()` / `IsEmpty()` | state |
| `IsValidSlot(Slot)` / `IsSlotOccupied(Slot)` | slot checks |
| `CanAcceptItem(Def)` | whether this type is accepted at all |
| `CanAcceptItemCount(Def, Desired)` | **how many will actually fit right now** |
| `CanSlotAcceptItem(Slot, Def)` | whether a specific slot will accept it |
| `HasContainerAuthority()` | whether this side has authority (shared by inventory and equipment) |
| `CanAcceptRequestFrom(Requester)` | **whether this actor is allowed to change the container** |

### Serialization

`ExportState()` → `FISDemoInventorySaveData` · `ImportState(State)` *(server)*

### Events

| Event | When |
|---|---|
| `OnInventoryChanged` | any change (with the change type) |
| `OnItemAdded` | a slot received an item |
| `OnItemRemoved` | a slot emptied |
| `OnItemChanged` | the item is the same, the data changed |
| `OnAddRejected` | an add was refused, **with the reason text** |

---

## `UISDemoEquipmentComponent`

### Configuration

| Field | Description |
|---|---|
| `AvailableSlots` | the body plan — which slots exist at all |
| `LinkedInventory` | the backpack items are taken from and returned to |
| `bAutoLinkInventoryOnSameActor` | find the inventory on the same actor (on by default) |

### Actions

| Method | Description |
|---|---|
| `TryEquipFromInventorySlot(Index)` | equip whatever is in an inventory slot |
| `TryEquipItem(Item, Slot, bRemoveFromInventory)` | equip a specific item |
| `TryUnequipSlot(Slot, bReturnToInventory)` | unequip |
| `TrySwapSlots(A, B)` | swap places *(requires `AlternativeSlots` on the items)* |
| `TryUnequipAll(bReturnToInventory)` | unequip everything |
| `TryUseEquippedItem(Slot, Instigator)` | use a worn item |

### Queries

| Method | Description |
|---|---|
| `GetEquippedItem(Slot)` | the item in the slot, or `null` |
| `GetEquippedActor(Slot)` | the slot's visual actor |
| `IsSlotEquipped(Slot)` / `HasSlot(Slot)` | state and slot presence |
| `GetAllEquippedItems()` / `GetOccupiedSlots()` / `GetAllEntries()` | iteration |
| `GetEntriesView()` | iteration without an array copy *(C++ only)* |
| `CanEquipToSlot(Item, Slot)` | whether it can be equipped here |
| `GetDesiredSlotForItem(Item)` | where the item asks to go |
| `GetTotalStatValue(StatTag)` | **the aggregate stat across everything worn** |
| `GetLinkedInventory()` | the linked backpack; if the cache is empty — looks on the actor |
| `HasContainerAuthority()` | whether this side has authority (shared by inventory and equipment) |

### Serialization and events

`ExportState()` / `ImportState(State)`

| Event | When |
|---|---|
| `OnItemEquipped` | a slot **became** occupied — the equip itself, not any change |
| `OnItemUnequipped` | a slot freed up |
| `OnEquippedItemChanged` | the item is the same, its data changed (durability, charges) |
| `OnEquipmentActorSpawned` | the slot's visual actor was created and attached |

> The first three events are exactly the same trio as the inventory's
> (`OnItemAdded` / `OnItemRemoved` / `OnItemChanged`). Bind the equip sound to
> `OnItemEquipped`: it won't fire again just because the sword lost a point of
> durability.

---

## `UISDemoItemDefinition` — the item type

| Field / method | Description |
|---|---|
| `DisplayName`, `Description`, `Icon`, `PickupMesh` | presentation |
| `ItemTags` | classification |
| `Fragments` | capabilities |
| `FindFragmentByClass` / `HasFragment` / `GetAllFragments` | fragment access |
| `HasAnyTags` / `HasAllTags` | tag checks |
| `GetMaxStackSize()` | the stack size (chained through the fragments) |
| `GetUnitWeight()` | the unit weight |
| `IsUsable()` | whether "use" makes sense |

---

## `UISDemoItemInstance` — a concrete copy

| Field / method | Description |
|---|---|
| `Definition`, `StackCount`, `StatValues` | state |
| `GetStatValue` / `SetStatValue` / `ModifyStatValue` | stats |
| `HasStat` / `ClearStatValue` | presence and removal |
| `CanStackWith(Other)` | whether it can merge |
| `GetMaxStackSize` / `IsStackFull` / `GetAvailableStackSpace` | stack |
| `GetDisplayName` / `GetDescription` | presentation |
| `GetStackWeight()` / `IsUsable()` | derived |
| `DuplicateInstance(Outer)` | a copy preserving state |
| `GetOwningContainer()` | **who holds it — the inventory or equipment** |
| `GetOwningInventory()` | the inventory, if it's that; `null` when the item is worn |
| `ConsumeFromContainer(Count)` | ask whoever holds it to spend copies |

---

## Fragments — their own helpers

Each fragment's fields are described in [04 — Fragments](04-Fragments.md).
Besides fields, some fragments expose functions you can call directly:

| Fragment | Function | Description |
|---|---|---|
| `UISDemoFragment_Durability` | `GetCurrentDurability(Instance)` | the copy's current durability |
| | `GetDurabilityPercent(Instance)` | 0–1, ready for a progress bar |
| | `IsBroken(Instance)` | whether durability dropped to zero |
| | `Repair(Instance, Amount)` | repair; a negative value — fully *(server)* |
| `UISDemoFragment_Consumable` | `GetRemainingCharges(Instance)` | how many charges are left |
| `UISDemoFragment_Equippable` | `AcceptsSlot(Slot)` | whether the item agrees to live in this slot |

All twelve hooks a custom fragment overrides are also public API:
`OnInstanceCreated`, `OnAddedToInventory`, `OnRemovedFromInventory`,
`CanBeAddedTo`, `CanStackWith`, `CanBeUsed`, `IsUsable`, `OnUsed`, `OnEquipped`,
`OnUnequipped`, `ModifyMaxStackSize`, `ModifyUnitWeight`.

---

## `UISDemoInventoryQueryLibrary` — searching by description

For when the question is more complex than "how many of these do I have": "all
the weapons that are worn down", "everything sellable", "the first arrow stack
with more than ten".

| Function | Description |
|---|---|
| `QueryInventory(Inv, Query)` | every matching item |
| `QueryInventoryFirst(Inv, Query)` | the first match, or `null` |
| `QueryInventoryCount(Inv, Query)` | the total copy count across all matches |
| `QueryInventorySlots(Inv, Query)` | the slot indices of all matches, ascending |
| `MakeTagQuery(Tag)` | shortcut: everything with one tag |
| `MakeDefinitionQuery(Def)` | shortcut: exactly one item type |

`FISDemoInventoryQuery` fields (an empty query matches everything; fields combine
with AND):

| Field | Description |
|---|---|
| `RequiredTags` | the item must carry **all** of these tags |
| `AnyOfTags` | the item must carry **at least one** of these tags |
| `ExcludedTags` | an item with any of these tags is excluded |
| `RequiredFragment` | the item must have a fragment of this class (subclasses too) |
| `MinStackCount` / `MaxStackCount` | stack-size bounds |
| `SpecificDefinition` | exactly this item type |

---

## Interfaces — integration points

| Interface | For |
|---|---|
| `IISDemoCollectibleInterface` | make your own actor something `UISDemoInventoryComponent::TryCollect` can pick up. `AISDemoItemPickup` already implements it; your own — only if your "collectible" object isn't a subclass of it. One method: `TryGiveContents(Collector)`, called already on the server |
| `IISDemoInventoryInterface` | say which of several inventories on an actor is primary — `GetInventoryComponent`, `GetEquipmentComponent`, `GetAllInventoryComponents`. Rarely needed: `GetInventoryFor` finds the single one itself. *(C++ only)* |

---

## World objects (`InventorySystemDemoWorld`)

### `AISDemoLootContainer`

| Member | Description |
|---|---|
| `ContainerInventory` | the contents |
| `LootTable`, `bGenerateLootOnSpawn` | filling |
| `bUseWeightedLoot`, `WeightedDropCount` | "exactly N items" mode |
| `bSingleUse` | a one-time chest |
| `Open(Opener)` / `Close(Closer)` | interaction — **server-only**, don't forward themselves |
| `CanOpen()` / `IsEmpty()` | state |
| `ResetContainer(bClear)` | allow it to refill |
| `BP_OnOpened` / `BP_OnClosed` / `BP_OnLooted` / `BP_OnLootGenerated` | visual hooks |

### `AISDemoItemPickup`

| Member | Description |
|---|---|
| `ItemDefinition`, `ItemCount` | contents for level-placed pickups |
| `HeldItem` | the real object with state |
| `bDestroyOnCollected`, `LifeSpanSeconds` | behaviour |
| `TryCollect(Collector)` | pick up — **server-only**; from a client call `UISDemoInventoryComponent::TryCollect` |
| `HasContents()` / `GetPickupText()` | state and label |
| `SpawnForItem(World, Item, Location, Class)` | drop an existing item into the world *(server)* |

> **Which `TryCollect` to call.** The one on the player's inventory. The pickup
> has no network connection, so it can't forward the request to the server
> itself — the player's inventory can. This is the same reason you take from a
> container through `TryTransferFrom` rather than the container's own methods.

### `UISDemoLootTable`

| Method | Description |
|---|---|
| `GenerateLoot()` | roll each entry separately by `DropChance` |
| `GenerateWeightedLoot(Count)` | exactly N entries by `Weight` |
| `PopulateInventory(Inv, bWeighted, Count)` | generate and place immediately |

---

## `UISDemoInventoryWorldSubsystem` — world-level state

Everything shared for the game but **not** for the process: subscriptions and
timers that must disappear with the world. Get it via
`UISDemoInventoryWorldSubsystem::Get(WorldContext)`.

| Member | Description |
|---|---|
| `GetPlayerInventory(PlayerState)` | find a player's inventory, whether on the `PlayerState` or the pawn |
| `GetPlayerEquipment(PlayerState)` | the same for equipment |
| `OnConsumableUsed` | **the consumable-used event** — the item, who used it, `EffectTags` |
| `GetFragmentCooldownRemaining(Fragment)` | how many seconds are left on a fragment's shared cooldown |
| `StartFragmentCooldown(Fragment, Seconds)` | start it |

> Why not a static on the fragment class: the fragment is shared by the whole
> game, and a static on it would be shared by every world in the process too.
> In PIE the server and client worlds would hear each other's uses, and
> subscriptions from a previous session would stay alive into the next.

The cooldown is keyed by **fragment**, so it's one per world for every player.
This is a deliberately simple tool — build per-player cooldowns on top of
`OnConsumableUsed`.

---

## Debugging (`InventorySystemDemoDebug`)

| Function | Description |
|---|---|
| `ToggleInventoryDebugOverlay(PC)` | open / close the inspector |
| `IsInventoryDebugOverlayOpen(PC)` | whether it's open now |

Both do nothing in Shipping. Console commands —
[13 — Debugging](13-Debugging.md).

---

## Enums

| Type | Values |
|---|---|
| `EISDemoItemUseResult` | `Success`, `NoItem`, `NotUsable`, `Blocked` |
| `EISDemoInventoryChangeType` | `ItemAdded`, `ItemRemoved`, `ItemChanged`, `SlotSwapped` |
| `EISDemoSortMode` | `ByName`, `ByStackCount`, `ByType`, `ByWeight` |
| `EISDemoInventoryHost` | `PlayerState`, `Pawn` |

## Structs

| Type | Description |
|---|---|
| `FISDemoItemStatEntry` | one per-instance value: tag + number |
| `FISDemoSlotRestriction` | a rule for a slot range |
| `FISDemoInventoryEntry` | an occupied slot: index + item |
| `FISDemoEquipmentEntry` | an occupied equipment slot: tag + item + actor |
| `FISDemoLootTableEntry` | a loot-table entry: item, count range, chance and weight |
| `FISDemoLootDrop` | a roll result: item + exact count |
| `FISDemoInventoryQuery` | a description of what to search an inventory for |
| `FISDemoItemSaveData` / `FISDemoInventorySaveData` / `FISDemoEquipmentSaveData` | serialization |

## Project settings

`UISDemoInventorySettings` — **Project Settings → Game → Inventory System**.
Everything is off or neutral by default; details in
[02 — Setup](02-Setup.md).

| Field | Description |
|---|---|
| `bAutoCreatePlayerInventory` | give every player an inventory automatically |
| `bAutoCreatePlayerEquipment` | and equipment along with it |
| `AutoInventoryHost` | where it lives: `PlayerState` (survives death) or `Pawn` |
| `DefaultInventorySlots` | how many slots an automatic inventory gets |
| `DefaultEquipmentSlots` | the body plan for automatic equipment |
| `InventoryComponentClass` / `EquipmentComponentClass` | your subclasses instead of the defaults |
| `MaxInteractionDistance` | server-side distance limit for requests; 0 — don't check |
| `bValidateItemsOnStartup` | validate item assets at editor startup |
| `bVerboseLogging` | detailed log of every inventory change |
