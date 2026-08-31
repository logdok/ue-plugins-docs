# Release Notes

*🇬🇧 English | [🇺🇦 Українська](../uk/Release-Notes.md)*

A short note on each release: what's new, what's fixed, what to watch for when
upgrading. The newest release is at the top.

---

> **This is the demo edition.** Every feature works exactly like the full plugin;
> only the volume of use is limited — a budget of 5 distinct item types per play
> session, an on-screen watermark, and no activation in Shipping builds. See
> [Demo Limitations](00-Demo-Limitations.md) for the details.

## 1.0

The first release. Unreal Engine **5.7 and 5.8**.

### What the plugin can do

**Items as data.** An item type is a Data Asset with a list of fragments. Five
shipped fragments: `Stackable`, `Equippable`, `Consumable`, `Durability`,
`Weight`. Your own are written in Blueprint or C++ and are no more limited than
the built-in ones.

**Inventory.** Slots or an unlimited mode, a weight limit, tag filters,
restrictions on individual slot ranges, a starting kit. Add, remove, move
between inventories, split stacks, quick move, sort, compact stacks.

**Equipment.** Slots are set by tags — your project defines the body plan. Visual
actors are created locally and cost no traffic. Aggregate stats across
everything worn.

**World objects.** Loot tables with two roll modes, containers with lazy
filling, items on the ground that keep their state when dropped and picked up.
Pickup is one `TryCollect` call on the player's inventory, which finds its own
way to the server.

**Examples included.** Seven ready Data Assets in `Content/Demo/Items` — a
sword, shield, helmet, torch, potion, ore and relic. Together they cover all
five fragments and every typical combination; pick them apart or delete them.

**Multiplayer.** One API for both modes: in single-player calls run in place and
create no RPC, on a client they forward themselves to the server. Replication
via `FFastArraySerializer` — one slot changes, one slot is sent.

**Serialization.** `ExportState` / `ImportState` hand you plain structs to embed
in your `USaveGame`. Item types are written as paths, so a save survives a
restart and patches.

**Debugging.** An interactive inspector (`DemoInvOverlay`) — it not only shows the
contents but lets you grant, use, equip and remove items right from the panel.
Plus 15 console commands, a detailed log, and asset validation at startup. All
disabled in Shipping.

### Architecture

The plugin is split into three modules:

| Module | Purpose |
|---|---|
| `InventorySystemDemo` | core; **no Slate/UMG dependency** |
| `InventorySystemDemoWorld` | chests, items on the ground, loot tables |
| `InventorySystemDemoDebug` | the inspector and commands; the only module with UI |

A project that only needs inventories on characters can skip `World`
altogether. A dedicated server never links the UI.

---

## Known limitations

Things the plugin deliberately **doesn't** do — so you don't spend time looking
for them.

**No ready UI.** The plugin gives data and events; you draw the grid, tooltips
and drag-and-drop yourself. No two games show an inventory the same way, so a
built-in widget would end up being worked around. How to do it —
[08 — Building the UI](08-Building-UI.md).

**No effect system.** The `Consumable` fragment reports the **tags** of what was
consumed, and what "heal 50" means is decided by your game. This is an
integration point with the Gameplay Ability System or your own system.

**No grid with item shapes.** Slots are uniform; an item takes exactly one slot
whether it's a sword or a ring. A "Tetris" inventory like Diablo or Escape from
Tarkov isn't supported.

**No built-in interaction system.** Containers and pickups have no mesh or
collision: a game with a trigger volume, a game with a trace and a game with
cursor-click want different things. Add your own and call `TryCollect` on the
player's inventory (for items on the ground) or `Open` on the container from the
server side of your interaction.

**No currency as a separate entity.** The simplest way is to make it an ordinary
item with a big stack — [11 — Recipes](11-Recipes.md#currency-as-an-item).

**The inventory isn't private.** Contents replicate to all observers of the
owning actor, not just the owner. This follows from the same component serving
both a player's backpack and a chest whose contents everyone should see.

**The consumable cooldown is simple.** Shared per item type, with no categories
and no persistence between sessions. Build more complex schemes on the
`OnConsumableUsed` event.

---

## Compatibility

The plugin works with **UE 5.7 and 5.8**.

It doesn't require particular base classes (`Character`, `PlayerState`,
`GameMode`), a particular input system or project structure, so it adds to an
existing project with no rework.

**Both** of UE's subobject replication mechanisms are supported — the plugin
follows the one your project chose and forces nothing.
