# 09 — Multiplayer

*🇬🇧 English | [🇺🇦 Українська](../uk/09-Multiplayer.md)*

The plugin's main promise: you write the code once, and it works in
single-player as well as multiplayer. This chapter explains how that's built,
and where the one exception is.

---

## One method instead of two

```cpp
Inventory->TryAddItem(Definition, 3, Remainder);
```

Internally there's just one check:

```cpp
bool UISDemoInventoryComponent::TryAddItem(...)
{
    if (!HasContainerAuthority())
    {
        Server_AddItem(Definition, Count);   // client -> server
        return true;                          // "request sent"
    }
    return AddItem_Internal(...);             // we have authority - work in place
}
```

| Situation | `HasAuthority()` | What happens |
|---|---|---|
| Single-player | `true` | runs in place, **no RPC is created** |
| Listen server (host) | `true` | the same, in place |
| Client | `false` | forwarded to the server, result via replication |

**Single-player serialises not a single byte.** The multiplayer capability
costs it nothing.

---

## The return value on a client

A client **can't** learn the result synchronously — the server hasn't answered
yet. So:

> On a client `Try*` returns `true` meaning **"request sent"**, not
> "succeeded".

```cpp
// WRONG - on a client this is always true
if (Inventory->TryAddItem(Def, 1, Rem))
{
    ShowPickupAnimation();   // fires even when the server refuses
}

// RIGHT - react to the event that arrives with replication
Inventory->OnItemAdded.AddDynamic(this, &UMyWidget::HandleItemAdded);
```

The interface is always built on events: `OnInventoryChanged`, `OnItemAdded`,
`OnItemRemoved`, `OnItemChanged`, `OnAddRejected`. They fire on both the server
and the clients, so the same handler serves both modes.

---

## The exception: ownerless containers

This is the **only** place where the networked nature leaks into the API.
Better to understand it right away than to fight the consequences.

**The problem.** A Server RPC only works when the actor has a network
connection through which the request can reach the server. The player's
character has one. A chest in the middle of a level doesn't. The engine
silently drops such a call.

```cpp
// DOESN'T WORK on a client: the chest has no connection
ChestInventory->TryRemoveItemFromSlot(0, 1);
```

**The solution.** The request always goes through the **player's** inventory,
which does have a connection. The server, on receiving it, authoritatively moves
items in both inventories:

```cpp
// WORKS everywhere
MyInventory->TryTransferFrom(ChestInventory, SlotIndex);   // take
MyInventory->TryTransferTo(ChestInventory, SlotIndex);     // put
```

In Blueprint there's a node that picks the direction itself:

```
Move Item Between Actors (From Actor, To Actor, From Slot)
```

> **A rule worth remembering:**
> any interaction with a world container is initiated by the **player's
> inventory**.

**Items on the ground — the same principle, but it's already done for you.** A
pickup has no connection either, so instead of `Pickup->TryCollect(Player)`
call:

```cpp
// WORKS everywhere: the route goes through the player's inventory
MyInventory->TryCollect(Pickup);
```

This is an ordinary `Try*` — it forwards itself from a client.
`AISDemoItemPickup::TryCollect` also exists, but it's a low-level server call that
the previous one ends up in.

### What stays server-side

Three things don't route themselves, because they have nothing to route
through:

| Call | What to do instead |
|---|---|
| `AISDemoLootContainer::Open` / `Close` | call from the **server** side of your interaction |
| `AISDemoItemPickup::SpawnForItem` | dropping an item is a Server RPC on the dropping player |
| `UISDemoInventoryComponent::TryAddItemInstance` | a client can't pass the server an object pointer; use `TryTransferFrom` or `TryCollect` |

All three refuse on a client and log — nothing breaks silently.

---

## What stops a client asking for too much

The scheme above lets a client say "move slot 3 of **that** inventory into
mine". Nothing in the shape of that sentence would stop it from naming another
player's backpack — so the server takes care of that.

Every `Server_*` RPC, before doing any work, passes through
`UISDemoItemContainerComponent::CanAcceptRequestFrom`. Who sent the request is taken
from the connection the RPC arrived on, not from an argument — the argument is
exactly what a modified client controls entirely.

When the request names a **second** container, it's checked only if items move
**out of it**. The asymmetry is deliberate: taking from someone else's
container is theft, putting into one is at worst a nuisance, and you can only
give away what's yours. Requiring ownership of the destination too would break
handing an item to another player, which the plugin itself cites as an example.
A game that doesn't want unsolicited gifts overrides `CanAcceptRequestFrom` on
the receiving side.

The default implementation accepts the request if one of two things holds:

- the container belongs to the same player who sent the request (`UNetConnection`s
  are compared, so a controller, pawn and `PlayerState` are all equally valid);
- the container has **no connection at all** — it's a world container (a chest,
  a corpse, a vending machine), and it's open to anyone who reached it;

and additionally — if the project settings define a `MaxInteractionDistance`,
the requester is within that distance of the container's owner.

```ini
[/Script/InventorySystemDemo.ISDemoInventorySettings]
; A server-side sanity limit, not an interaction range: set it several times
; larger than your real range so ordinary latency never trips it.
; 0 (the default) disables the check.
MaxInteractionDistance=1000.0
```

Override `CanAcceptRequestFrom` for anything your game adds on top: a locked
chest, a shop with opening hours, a corpse only its killer may search.

> The check runs **only on requests that crossed the network**. Calls on the
> server and every call in single-player go straight to the work.

---

## What replicates and how

### Inventory contents

`FISDemoInventoryList` is built on `FFastArraySerializer`: one slot changes — one
slot is sent, not the whole inventory. The difference between 40 bytes and a
few kilobytes when a player picks up a pebble.

Items (`UISDemoItemInstance`) are replicated UObject subobjects. The plugin supports
**both** of UE's subobject replication mechanisms and follows the one your
project chose:

- the classic `ReplicateSubobjects` (default in UE 5.7);
- the subobject registry, if the project turned on
  `net.SubObjects.DefaultUseSubObjectReplicationList`.

The plugin deliberately does **not** force the registry on: it only works when
the owning actor turned it on too, and forcing it on a component dropped onto an
"ordinary" actor would silently break item replication.

### Inventory capacity

`MaxSlots` and `MaxWeight` replicate along with the contents. This matters
exactly when capacity **changes at runtime**, and here's why.

A value set in Blueprint reaches clients on its own — it lives in the class
defaults, and a client creates the component already carrying it. But a runtime
assignment doesn't change the defaults: the server does
`Inventory->MaxSlots = 40`, and without replication a client would stay with the
number the component was born with.

The consequences of such a mismatch are purely client-side, but visible:
`IsFull()`, `GetFreeSlotCount()`, `IsValidSlot()` and `CanAcceptItemCount()` on
a client would answer for the old size, the grid would draw the wrong number of
cells, and dragging into slot 35 would be highlighted as forbidden. The items
themselves are never lost — every mutation is server-side, and the server always
counts correctly.

This applies to two perfectly ordinary scenarios:

- **a backpack upgrade** — a perk, a quest reward, a new bag;
- **auto-creation of components** — the subsystem applies
  `DefaultInventorySlots` on the server, i.e. also at runtime.

### Equipment

Also via `FFastArraySerializer`. **The visual actor is not replicated** — each
machine creates its own from the `Equippable` fragment. So a sword in the hand
costs exactly as much as the slot entry.

### Who sees what

Contents replicate to **all** observers of the owning actor, not just the
owner. This is a deliberate compromise: the same component class serves both a
player's backpack and a world chest, whose contents everyone should see.

If you need strict privacy for a player's inventory, wrap access in your own UI
— the plugin doesn't hide data at the replication level.

---

## Single-player and multiplayer on one map

Nothing to switch. Components are created where the world has authority: in
single-player that's the one world, on a network it's the server, and clients
get replicas.

No branch of your code has to ask which mode it's in.

---

## Automatic component attachment

`UISDemoInventoryWorldSubsystem` can create components for every player itself. Off
by default — see
[02 — Setup](02-Setup.md#automatic-component-creation).

One mechanism covers login, late join, respawn, seamless travel and PIE.
Manually placed components always take precedence.

Where exactly to create them:

| `Auto Inventory Host` | When appropriate |
|---|---|
| `PlayerState` | the inventory survives death and respawn — a typical backpack |
| `Pawn` | the inventory is lost with the body — extraction shooters, roguelikes |

---

## Testing in multiplayer

In the editor: **Play → Net Mode → Play As Client**, number of players ≥ 2.

Console commands work on a client too — they forward themselves to the server,
and the report comes back:

```
DemoInvGive Potion 5
DemoInvList
DemoInvOverlay
```

The inspector shows the net mode with a coloured badge: green — the
authoritative side, yellow — a client. Open it on both sides to see whether the
state matches.

If an item didn't appear:

```
Log LogDemoInventorySystem Verbose
```

Prints every add, remove and move with slot indices.

---

## Where to next

- Persist state between sessions: [10 — Serialization](10-Serialization.md)
- Diagnostic tools: [13 — Debugging](13-Debugging.md)
