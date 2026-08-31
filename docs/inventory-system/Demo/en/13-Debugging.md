# 13 — Debugging

*🇬🇧 English | [🇺🇦 Українська](../uk/13-Debugging.md)*

Everything in this chapter is provided by the `InventorySystemDemoDebug` module. It
works as soon as the plugin is enabled and is **completely disabled in Shipping
builds**.

---

## The inspector

An overlay panel with the live state of the local player's inventory and
equipment.

```
DemoInvOverlay
```

Or from code — to bind it to a key:

```cpp
UISDemoInventoryDebugLibrary::ToggleInventoryDebugOverlay(PlayerController);
```

### What it shows

**Header** — the owning actor, the net mode, and whether this side has
authority. The mode is colour-coded: green — the authoritative side, yellow — a
client.

This is the most useful thing to see right away: most confusing networked
inventory behaviour comes from looking at a client and expecting authoritative
results.

**Summary** — occupied slots and weight with bars that change colour from green
to yellow to red as they fill.

**Contents** — one row per occupied slot:

| Row element | What it means |
|---|---|
| blue number | the **real** slot index, including gaps |
| name and `x5 / 20` | how many in the stack and how many fit |
| grey line | the item's fragments: `Stackable, Durability` |
| purple **UNIQUE** badge | the item has its own values — **which is why it doesn't stack** |
| values next to it | the values themselves: `Durability 42` |

**Equipment** — one row per slot in `AvailableSlots`, **including empty ones**.
An empty slot is information too.

**Buttons** — `Sort`, `Compact`, `Clear`.

### What questions it answers

| Question | Where the answer is |
|---|---|
| "Why don't these two items merge?" | the **UNIQUE** badge and the values next to it |
| "Why did the item land in slot 7?" | slot indices are shown literally |
| "Why won't it equip?" | the fragment list (is there an `Equippable`) and the character's slot list |
| "Does the client see the same thing?" | the net-mode badge in the header |
| "Why won't it fit?" | the slot and weight bars |

### Notes

- **The game doesn't pause** — the state changes while you look at it.
- **Updates itself** — on events plus a slow timer.
- **Needs no asset** — the widget tree is built in C++.
- **Never opens in Shipping**, whoever calls it.

> The most useful way to use it: open the inspector **on the server and on the
> client at the same time** (`Play As Client`, 2 players) and compare. A
> discrepancy between them is almost always the root of the problem.

---

## Console commands

Available automatically on any `CheatManager`. They work on a client too:
commands that change state forward themselves to the server, and commands that
show state run there and send the report back.

### Inventory

| Command | Effect |
|---|---|
| `DemoInvOverlay` | open / close the inspector |
| `DemoInvItems [name part]` | list the item types in the project |
| `DemoInvGive <name part> [count]` | give yourself an item |
| `DemoInvRemove <name part> [count]` | take an item away from yourself |
| `DemoInvClear` | clear the inventory |
| `DemoInvFill [count]` | fill with random items |
| `DemoInvList` | show the contents |
| `DemoInvUse <slot>` | use an item |
| `DemoInvSort` | sort by name |
| `DemoInvCompact` | merge partial stacks |
| `DemoInvWeight` | show weight against the limit |

### Equipment

| Command | Effect |
|---|---|
| `DemoEquipList` | show what's worn, including empty slots |
| `DemoEquipSlot <inventory slot>` | equip the item from that slot |
| `DemoEquipRemove <slot tag>` | unequip a slot |
| `DemoEquipClearAll` | unequip everything |

Item search is by **name part**, case-insensitive: `DemoInvGive sword` finds
`DA_IronSword`. An exact asset-name match takes priority, so an unambiguous
query isn't hijacked by a longer name.

**Start with `DemoInvItems`** — it shows which items exist in the project at all,
and how to name them in the other commands.

---

## Verbose log

```
Log LogDemoInventorySystem Verbose
```

Prints every add, remove and move with slot indices and stack sizes. Usually
that's enough to see why an item went somewhere else.

Turn it on permanently: **Project Settings → Game → Inventory System → Verbose
Logging**.

What to look for in the log:

| Line | Means |
|---|---|
| `Add refused ('X'): ...` | the item was refused, and why |
| `AddItem 'X' x5 -> remainder 2` | not everything fit |
| `Added [Slot 3] X x5` | what actually landed and where |
| `requires authority` | you called a server method on a client |

---

## Item validation

In dev builds the plugin validates every `ISItemDefinition` at world startup and
writes problems to the log: empty names, duplicate fragments, an `Equippable`
with no slot, contradictory loot-table bounds.

Turn it off: **Project Settings → Game → Inventory System → Validate Items On
Startup**.

The same checks run in the editor on asset save — problems show right on the
asset.

---

## Automated tests

**Tools → Session Frontend → Automation**, filter `InventorySystemDemo.`

Or from the command line:

```bash
UnrealEditor-Cmd Project.uproject \
  -ExecCmds="Automation RunTests InventorySystemDemo.; Quit" \
  -unattended -nullrhi
```

Groups: `InventorySystemDemo.Inventory`, `.Equipment`, `.Loot`, `.Persistence`,
`.ItemDefinition`, `.ItemInstance`, `.DebugOverlay`.

---

## The "why isn't it working" checklist

Work top to bottom — the questions are ordered by frequency.

**An item won't add.**
1. `DemoInvList` — was the inventory found at all? If "you have no inventory", the
   component isn't on the actor.
2. Subscribe to `OnAddRejected` — it carries a ready reason.
3. `DemoInvWeight` — is it hitting the weight limit?
4. Check `AllowedItemTags` / `BlockedItemTags` on the component.

**Items won't stack.**
1. Open the inspector: is there a **UNIQUE** badge? Then the item has its own
   values — that's the reason.
2. Is there a `Stackable` fragment? Without it the stack max is 1.

**An item won't equip.**
1. Inspector: is `Equippable` in the fragment list?
2. `DemoEquipList`: does the character have the slot in `AvailableSlots`?
3. The slot in the fragment and the slot in the component must be the **same**
   tag.

**Works in the editor, not on a client.**
1. Are you relying on the `Try*` return value? On a client it means "sent".
2. Are you accessing the chest directly? Go through the player's inventory —
   [09 — Multiplayer](09-Multiplayer.md#the-exception-ownerless-containers).
3. Open the inspector on both sides and compare.

**The UI doesn't update.**
1. Subscribed to events, or polling every frame?
2. Did the subscription happen after the component appeared? On a client it can
   arrive later than the widget.

**Tags aren't found.**
1. Check the spelling — tags are case-sensitive.
2. Add your own tags to your project's **own** `Config/Tags/*.ini`, not the
   plugin's file: a plugin update would overwrite it.

---

## Where to next

- Get to grips with the network: [09 — Multiplayer](09-Multiplayer.md)
- Short answers to common questions: [14 — FAQ](14-FAQ.md)
