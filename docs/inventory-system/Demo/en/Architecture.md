# Architecture

*🇬🇧 English | [🇺🇦 Українська](../uk/Architecture.md)*

This page is for those who read the code: extending the plugin, embedding it
deeply, or evaluating it before adopting it.

The rest of the guide answers "how do I use this". Here it's "how is it built,
and why this way", along with decisions that look odd until you know the reason.

---

## Three modules

```
InventorySystemDemo       core: data, replication, components, queries
InventorySystemDemoWorld  chests, items on the ground, loot tables
InventorySystemDemoDebug  the inspector and console commands
```

**The core doesn't depend on Slate, UMG or anything visual.** This isn't a
stylistic requirement: a dedicated server shouldn't link the UI just to know
what's in a backpack. The only module allowed UI is `Debug`, and it's
completely disabled in Shipping.

**World is split out** so a project that only needs inventories on characters (a
card game, a strategy title, a UI-only crafting shell) doesn't compile actors it
will never place.

---

## The central idea: authority routing

Every method that changes state is public, named `Try*`, and looks like this:

```cpp
bool UISDemoInventoryComponent::TryAddItem(...)
{
    if (!HasContainerAuthority())
    {
        Server_AddItem(Definition, Count);   // client -> server
        return true;                          // "request sent"
    }
    return AddItem_Internal(...);             // we have authority - work here
}
```

`Server_*` RPCs and `*_Internal` implementations are `protected`. They are **not
part of the contract**: a plugin user doesn't see them and doesn't choose
between them.

### Why not two methods

The familiar alternative is a pair of methods: "send a request" and "do it
authoritatively". It forces everyone who writes a call to remember exactly where
it runs, and punishes a mistake silently: an authoritative method marked
`BlueprintAuthorityOnly`, called from a client UI widget, simply does nothing —
with no error and no warning.

One method removes this class of bugs and **costs single-player nothing**:
there the owner is always authoritative, the `if` branch doesn't fire, no RPC is
created.

### The price of the decision

On a client `Try*` returns `true` meaning "request sent", not "done" — knowing
the result synchronously is impossible. So the UI is built on events. This is
documented in every tooltip and in [09 — Multiplayer](09-Multiplayer.md).

### The one exception

A chest in the world has no network connection, so a Server RPC through its
component is dropped by the engine. Interaction with containers goes through the
**player's** inventory (`TryTransferFrom` / `TryTransferTo`). This isn't a
workaround but the correct form: the client asks its own component, the server
authoritatively moves items in both.

### The request gate

The flip side of the same form: if a client can name **someone else's**
container as a transfer source, something has to stop it — and it can't be the
caller, because the whole point of the scheme is that nobody writes networking
code.

So every `Server_*` RPC starts with `ValidateIncomingRequest`, which asks
`CanAcceptRequestFrom` of the target container and — if the request names a
second one — of that one too. Who sent the request is taken from the connection
the RPC arrived on, not from an argument: the argument is the one thing a
modified client controls entirely.

Details and examples are in [09 — Multiplayer](09-Multiplayer.md).

---

## The shared container base

Items are stored by two components: `UISDemoInventoryComponent` (slots by number)
and `UISDemoEquipmentComponent` (slots by tag). The difference between them is slot
addressing, and almost nothing else. The shared part is lifted into
`UISDemoItemContainerComponent`:

| What | Why exactly there |
|---|---|
| `HasContainerAuthority()` | one authority check instead of two copies |
| `CanAcceptRequestFrom()` | the request gate, identical for both |
| `RemoveHeldInstance()` | "spend me" — the item asks whoever holds it |
| `NotifyItemInstanceChanged()` | "my data changed" — flag the slot for replication |
| subobject registration | identical for both, easy to forget in one |

This isn't cosmetic. The key consequence: `UISDemoItemInstance::OwningContainer`
points at the **base**, not at an inventory. If it only knew the inventory,
then:

- a stat written on a worn item would notify nobody — the slot wouldn't be
  flagged for replication, and no panel would redraw;
- a fragment that wants to spend an item would have to look up its slot index in
  the inventory; a worn item has no such index, so a drunk worn flask would
  announce its effect and **spend nothing**.

Both properties belong to "whoever holds the item", so they live in one place.

---

## Fragments as the extension mechanism

`UISDemoItemDefinition` has no behaviour of its own — only a list of
`UISDemoItemFragment`s. The shipped five (`Stackable`, `Equippable`, `Consumable`,
`Durability`, `Weight`) are ordinary subclasses, no more privileged than yours.

### The rule that determines everything else

The fragment object lives **inside the asset** and is shared by **every**
instance of the item in the game. A hundred torches for a hundred players point
at one `UISDemoFragment_Durability` object.

So:

- every hook is declared `const`;
- writing to a fragment field at runtime edits the asset for everyone at once;
- mutable state belongs to `UISDemoItemInstance::StatValues`.

Violating this rule corrupts data globally and invisibly — it's the single most
important invariant in the codebase.

### How the core stays independent

The core **includes** no concrete fragment. Derived values are computed as a
chain:

```cpp
int32 Value = 1;                                   // initial
for (Fragment : Fragments)
    Value = Fragment->ModifyMaxStackSize(Value);   // each may change it
```

That's why an item without `Stackable` has a stack of 1 — nobody changed the
initial value. Adding a new kind of derived value = adding a hook, without
touching the components.

### Determinism

Query hooks (`CanStackWith`, `Modify*`) run **on clients too**. A
non-deterministic hook produces a desynchronised inventory. Hooks that change
state — server-only.

---

## Replication

### FastArraySerializer

`FISDemoInventoryList` and `FISDemoEquipmentList` send **deltas**: one slot changed —
one slot goes out. The difference between 40 bytes and a few kilobytes when a
player picks up a pebble.

### Two subobject paths — and why neither is forced

Items (`UISDemoItemInstance`) are replicated UObjects. UE has two mechanisms: the
classic `ReplicateSubobjects` and the subobject registry.

The plugin supports **both** and follows the one the project chose. The registry
can't be forced on: it only works when the **owning actor** turned it on too,
and the engine default is off. A component with a forced registry on an
"ordinary" actor would silently stop replicating items at all.

### The entry-pointer trap

`TArray::RemoveAt` shifts elements. An `FISDemoInventoryEntry*` taken before a
mutation points, after it, at a different entry — or past the end of the array.

So **all mutation works with slot indices**, not with pointers, and there's a
test for it (`InventorySystemDemo.Inventory.Swap.Regression.MergeDoesNotCorrupt`).
If you extend the plugin — this is the easiest rule to break by accident.

### When the owner is assigned

`FISDemoInventoryList::OwnerComponent` is assigned in the component's
**constructor**, not in `BeginPlay`. A client can receive the first delta before
`BeginPlay`, and with a null owner the FastArray callbacks would silently lose
the very first update — the classic "the UI is empty until you pick up a second
item" bug.

---

## Slots are sparse

Indices run `0..MaxSlots-1`, but internally only **occupied** entries exist. A
30-slot inventory with three items stores three entries with indices of, say, 0,
7 and 29.

Consequence for code: `GetAllEntries()[N]` is not slot N. The index is taken
from the entry itself (`Entry.SlotIndex`).

`MaxSlots = 0` means an unlimited inventory. There's no separate flag,
deliberately: two independent fields ("how many slots" and "unlimited?")
inevitably contradict each other the moment someone changes one and forgets the
other.

---

## Code map

| File | What to read |
|---|---|
| `Core/ISDemoInventoryComponent.h` | the class comment describes authority routing — it repeats everywhere |
| `Core/ISDemoItemContainerComponent.h` | what the inventory and equipment share, and the request gate |
| `Core/ISDemoItemFragment.h` | the fragment-sharing rule and all 12 hooks |
| `Core/ISDemoItemInstance.h` | what makes a copy unique, the stacking rules |
| `Core/ISDemoInventoryList.h` | why mutation goes through methods, not pointers |
| `Core/ISDemoInventorySettings.h` | what the project configures at all |
| `Fragments/ISDemoFragment_Durability.cpp` | the fullest example of a self-contained fragment |

The header comments are deliberately expansive: the tooltips a designer sees in
the editor and the explanations for a programmer are the same text.

---

## Deliberate limitations

Things the plugin **doesn't** do, so you don't go looking for them in the code:

**No ready UI.** The plugin gives data and events. No two games draw an
inventory the same way, so a built-in widget would end up being worked around.

**No effect system.** `Consumable` reports the tags of what was consumed; what
"heal 50" means is decided by the game. This is an integration point with GAS or
your own system.

**No grid with item shapes.** A slot is uniform, an item takes exactly one. A
"Tetris" inventory isn't supported.

**No built-in interaction.** Containers and pickups have no mesh or collision: a
game with a trigger, a game with a trace and a game with a click want different
things.

**The inventory isn't private.** Contents replicate to all observers of the
owning actor — a consequence of one class serving both a player's backpack and a
chest whose contents everyone should see. A client can read someone else's
inventory; **change** it — no, the request gate closes that off.

**The `Consumable` cooldown is one per world.** It's shared by all players, so
it's no good for a competitive game. It's a deliberately simple tool; per-player
cooldowns are built on top of the `OnConsumableUsed` event.

**Slot lookup is linear.** `FISDemoInventoryList` finds a slot by scanning the
entries. For a 30–40-slot backpack this is faster than any index; for an
unlimited container of tens of thousands of items — no. Sorting, compaction and
weight accounting are linear in the inventory size; only the scan itself stays
quadratic, and that's deliberate.

---

## If you change the plugin's code

An ordinary project build compiles the **editor** target, where many headers
arrive transitively. Packaging compiles `UnrealGame`, where they don't — so a
plugin that builds clean in the editor can fail to package over a single missing
`#include`.

If you added a header that relies on transitive includes, verify with
packaging:

```bash
RunUAT.sh BuildPlugin -Plugin="<path>/InventorySystemDemo.uplugin" \
  -Package=<temp dir> -TargetPlatforms=Mac -Rocket
```

Errors come out one per run — expect several rounds.

---

## Where to next

- The data model in plain terms: [01 — Core Concepts](01-Core-Concepts.md)
- Write your own fragment: [04 — Fragments](04-Fragments.md#a-custom-fragment)
- Ready scenarios: [11 — Recipes](11-Recipes.md)
