# 07 — Loot and world items

*🇬🇧 English | [🇺🇦 Українська](../uk/07-Loot-And-Pickups.md)*

Everything in this chapter is the `InventorySystemDemoWorld` module. It's what
physically exists in a level: chests, items on the ground, loot tables.

A project that only needs inventories on characters can leave this module
untouched — it pulls no dependencies into the core.

---

## Loot table

An asset that describes **what** can drop. Created as a
**Data Asset → ISLootTable**.

Each entry:

| Field | Description |
|---|---|
| `Item` | what can drop |
| `MinCount` / `MaxCount` | the count range (equal values = fixed) |
| `DropChance` | 0–1 chance that this entry drops at all |
| `Weight` | relative weight when picking "N items" |

### Two ways to roll the dice

The same asset supports both typical shapes — it depends on which method you
call:

| Method | Behaviour | When to use |
|---|---|---|
| `GenerateLoot()` | rolls **every** entry separately by its `DropChance` | a chest with variable contents: any combination can drop |
| `GenerateWeightedLoot(N)` | picks **exactly N** entries by `Weight` | a boss that always drops two things; rare items have low weight |

`DropChance` is ignored in weighted mode, `Weight` in the ordinary one.

### Count limits

| Field | Effect |
|---|---|
| `MinItemsToSpawn` | if the rolls gave fewer — extra entries are drawn by weight |
| `MaxItemsToSpawn` | the excess is dropped; 0 = no ceiling |

`MinItemsToSpawn` guarantees a chest is never disappointing.

### Fill an inventory in one call

```cpp
LootTable->PopulateInventory(ChestInventory);
```

Returns how many drops were placed. What didn't fit goes into the log as a
warning — so a too-small chest is easy to spot during development.

---

## A chest, a crate, a corpse

`AISDemoLootContainer` — an actor with an inventory and optional filling from a
table.

### It deliberately has no mesh or collision

The plugin doesn't impose its own interaction system: a game with a trigger
volume, a game with a line trace and a game with cursor-click want different
things, and none of them wants it guessed for them.

Make a Blueprint from **ISLootContainer**, add the mesh and collision you need,
and call `Open` when your interaction fires.

> **`Open` and `Close` are server-only, and don't forward themselves.** This is
> the only place in the plugin where that's true: a chest has no network
> connection of its own, so a route from a client through it doesn't exist at
> all (the same reason items are taken from it through the player's inventory).
> Call them from the **server side** of your interaction — the player's
> interaction component has a connection, and its Server RPC is exactly for
> this.
>
> Calling from a client breaks nothing: it's refused and logged. But the chest
> stays un-rolled — and therefore empty, for whoever is looking at it from a
> client.
>
> Taking items after opening no longer needs this care: `TryTransferFrom`
> routes itself.

### Settings

| Field | Description |
|---|---|
| `LootTable` | what to fill with (empty = set contents by hand via `StartingItems`) |
| `bGenerateLootOnSpawn` | roll the dice immediately, not on first open |
| `bUseWeightedLoot` / `WeightedDropCount` | "exactly N items" mode |
| `bSingleUse` | a one-time chest: once emptied, it isn't refilled |

### When the dice are rolled

**By default — on first open**, not on spawn. This is deliberate:

- a hundred chests on a level cost nothing until players reach them;
- two players who open the same chest **always see identical contents** — the
  roll happened once, on the server.

Turn on `bGenerateLootOnSpawn` if the contents must be known in advance: a
minimap showing chest contents, or a test that checks them.

### Visual hooks

| Blueprint event | When |
|---|---|
| `BP_OnOpened` / `BP_OnClosed` | lid animation, a creak, a highlight |
| `BP_OnLooted` | the chest emptied: dim the glow, swap the mesh |
| `BP_OnLootGenerated` | the dice were rolled — add something of your own (scaling to player level, a guaranteed quest item) |

### Take an item from a chest

**Don't call the chest's inventory directly.** The request goes through the
player's inventory:

```cpp
PlayerInventory->TryTransferFrom(Container->GetContainerInventory(), SlotIndex);
```

Or the **Move Item Between Actors** node. Why this way —
[09 — Multiplayer](09-Multiplayer.md#the-exception-ownerless-containers).

### A chest that refills

```cpp
Container->ResetContainer(/*bClearExistingContents*/ true);
```

For a resource node that regenerates, or a trader who restocks overnight. Turn
off `bSingleUse` so the chest isn't marked as spent.

---

## An item on the ground

`AISDemoItemPickup` closes the "drop → pick up" loop.

### It preserves the item's state

Dropping is **not** "delete the item, remember the type". The pickup carries the
object itself, so a sword dropped at 12/100 durability is picked up at 12/100 —
with every enchantment and rolled value.

### Two ways to create one

**Placed in a level or created from a type** — set `ItemDefinition` and
`ItemCount`. A copy is created on the server as brand-new (full durability).

**From an existing item** — when a player drops something:

```cpp
// First take it out of the inventory, then hand it to the pickup.
UISDemoItemInstance* Item = Inventory->GetItemAtSlot(SlotIndex);
Inventory->TryRemoveItemFromSlot(SlotIndex, Item->StackCount);

AISDemoItemPickup::SpawnForItem(this, Item, DropLocation);
```

Both lines are **server-side**: the pickup is a replicated actor, so one created
on a client would be a ghost nobody else sees. `SpawnForItem` called from a
client refuses and logs. Dropping an item is a Server RPC on the dropping
player.

### Settings

| Field | Description |
|---|---|
| `bDestroyOnCollected` | destroy after pickup (default yes) |
| `LifeSpanSeconds` | lifetime; 0 = lies forever |

> `LifeSpanSeconds` is worth setting in any game where players drop things
> freely: a multi-hour session would otherwise accumulate thousands of
> abandoned actors.

### Pickup

Call it on the **collector's inventory**, not on the pickup itself:

```cpp
CollectorInventory->TryCollect(Pickup);
```

This is the same `Try*` as every other mutating method — call it from anywhere,
client or server, and it figures itself out. The reason: the pickup has no
network connection of its own (nobody "owns" it), so it can't forward a
client's request to the server itself. `UISDemoInventoryComponent` belongs to the
player and has a connection, so the request goes through it — the same
principle as `TryTransferFrom` for chests.

`AISDemoItemPickup::TryCollect(Collector)` (without the inventory) also exists, but
it's a low-level, **server-only** call — use it only from code that is
definitely already running on the server (GameMode, another Server RPC). From
client-side interaction code — Blueprint or C++ — always
`Inventory->TryCollect(Pickup)`.

You don't need to find the inventory yourself — it's all handled. Safe to call
repeatedly: a collected pickup refuses subsequent calls, so two players who
reached it at once won't each get a copy.

If there's no room — **the pickup stays in the world**, rather than being
destroyed along with its contents.

Visual hook: `BP_OnCollected`.

---

## Your own container without a ready class

If `AISDemoLootContainer` doesn't fit (your own base class, your own interaction
system), a container is simple to make: **it's an ordinary actor with an
inventory component**.

```cpp
UCLASS()
class AMyVendingMachine : public AMyInteractableBase
{
    GENERATED_BODY()

public:
    AMyVendingMachine()
    {
        bReplicates = true;

        Stock = CreateDefaultSubobject<UISDemoInventoryComponent>(TEXT("Stock"));
        Stock->MaxSlots = 12;
    }

    UPROPERTY(VisibleAnywhere)
    TObjectPtr<UISDemoInventoryComponent> Stock;
};
```

The player then takes from it the same way as from a chest — through their own
inventory. No inheritance from the plugin's classes required.

A corpse is made the same way: add an inventory component to the character and
don't delete the actor immediately on death.

---

## Where to next

- The direction rule for the network: [09 — Multiplayer](09-Multiplayer.md)
- Trading and crafting recipes: [11 — Recipes](11-Recipes.md)
