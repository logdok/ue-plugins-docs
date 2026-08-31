# 03 — Authoring Items

*🇬🇧 English | [🇺🇦 Українська](../uk/03-Authoring-Items.md)*

How to create an item type, a full reference of its fields, and ready recipes
for common items.

---

## Creating the asset

**Content Browser → right-click → Miscellaneous → Data Asset → ISItemDefinition.**

Location — anywhere in your content. The plugin discovers items through the
Asset Registry by class, so the folder structure is yours, and moving assets
breaks nothing.

> **One caveat about the name.** The asset name is the item's stable identity:
> it's what goes into saves and into console commands. Renaming the asset after
> release invalidates references in other people's save files. `DisplayName`, on
> the other hand, can be changed at any time — nothing references it.

---

## Field reference

### Presentation

| Field | Description |
|---|---|
| `DisplayName` | The name the player sees. Localised like any `FText`. |
| `Description` | Longer text for a tooltip or a details panel. |
| `Icon` | Icon for the UI. A **soft reference** — see below. |
| `PickupMesh` | Mesh for an item lying on the ground. Also a soft reference. |

**Why the icon is a soft reference.** Owning an item doesn't pull the texture
into memory. Your UI decides when to load it (Blueprint: `Async Load Asset`;
C++: `StreamableManager`). Because of this, a thousand item types don't drag a
thousand textures into the memory of a dedicated server that doesn't need them
at all.

### Classification

| Field | Description |
|---|---|
| `ItemTags` | What the item **is**: `Item.Type.Weapon.Melee`, `Item.Rarity.Rare`, `Item.Property.Tradeable`. |

These tags are read by inventory filters, slot restrictions and queries.
Typically an item has one type tag, one rarity tag and any number of property
tags.

Matching is **hierarchical**: an item tagged `Item.Type.Weapon.Melee` also
answers a query for `Item.Type.Weapon`.

### Capabilities

| Field | Description |
|---|---|
| `Fragments` | What the item **can do**. This is where item design happens. |

Order matters only for `OnUsed` — fragments run top to bottom.

Details on each fragment in [04 — Fragments](04-Fragments.md).

### Derived values (read-only)

These functions are computed from the fragments and are convenient to call from
the UI:

| Function | What it returns |
|---|---|
| `GetMaxStackSize()` | how many copies fit in a slot; `1` without a `Stackable` fragment |
| `GetUnitWeight()` | weight of one copy; `0` without a `Weight` fragment |
| `IsUsable()` | whether a "Use" button makes sense |

---

## Recipes

Ready combinations for the most typical items. Copy the closest one and change
the values.

### Health Potion

```
DA_HealthPotion
├── DisplayName: "Health Potion"
├── ItemTags: Item.Type.Consumable
└── Fragments:
    ├── Stackable      → MaxStackSize: 10
    └── Consumable     → ConsumeAmount: 1
                         EffectTags: Effect.Heal
```

The healing itself is yours to implement — the plugin only reports the effect
tags. How to wire it up is shown in
[04 — Fragments](04-Fragments.md#consumable--the-item-is-spent).

### Sword

```
DA_IronSword
├── DisplayName: "Iron Sword"
├── ItemTags: Item.Type.Weapon.Melee, Item.Rarity.Common
└── Fragments:
    ├── Equippable     → EquipmentSlot: Equipment.Slot.Weapon.Main
    │                    EquippedActorClass: BP_SwordActor
    │                    AttachSocketName: hand_rSocket
    │                    GrantedStats: { Item.Stat.Damage: 12 }
    └── Durability     → MaxDurability: 100
                         DurabilityPerUse: 1
```

Doesn't stack (no `Stackable`), has individual durability, appears in the
character's hand.

### Crafting material

```
DA_CopperOre
├── DisplayName: "Copper Ore"
├── ItemTags: Item.Type.Material
└── Fragments:
    ├── Stackable      → MaxStackSize: 99
    └── Weight         → UnitWeight: 0.5
```

### Ammo for a special slot

```
DA_Arrow
├── DisplayName: "Arrow"
├── ItemTags: Item.Type.Ammo
└── Fragments:
    └── Stackable      → MaxStackSize: 50
```

To make arrows land only in the first backpack slots, configure
`SlotRestrictions` on the component — see
[05 — Inventory Operations](05-Inventory-Operations.md#slot-restrictions).

### Quest item

```
DA_SealedLetter
├── DisplayName: "Sealed Letter"
├── ItemTags: Item.Type.Quest, Item.Property.QuestItem
└── Fragments: (empty)
```

No fragments: takes one slot, doesn't stack, does nothing when used. The
`Item.Property.QuestItem` tag lets a shop block its sale via `BlockedItemTags`.

### Torch with fuel

```
DA_Torch
├── DisplayName: "Torch"
├── ItemTags: Item.Type.Tool
└── Fragments:
    ├── Equippable     → EquipmentSlot: Equipment.Slot.Weapon.Offhand
    │                    EquippedActorClass: BP_TorchActor
    └── Consumable     → MaxCharges: 100
                         bDestroyWhenChargesSpent: true
```

Charges mode: the item stays in the slot spending charges, and disappears when
the fuel runs out.

---

## Item validation

The plugin validates assets automatically — in the editor on save, and at world
startup in dev builds. It reports:

- an empty `DisplayName`;
- an empty entry in `Fragments`;
- **two fragments of the same class** (the second silently overrides the first);
- an `Equippable` with no slot set;
- a `Stackable` with `MaxStackSize < 2`;
- a `Durability` with `MaxDurability <= 0` (every copy would start broken).

Turn it off at **Project Settings → Game → Inventory System → Validate Items On
Startup**.

To check everything by hand, run the `DemoInvItems` command — it lists every item
type in the project.

---

## Where to next

- Learn what each fragment can do: [04 — Fragments](04-Fragments.md)
- Learn to grant items to a player:
  [05 — Inventory Operations](05-Inventory-Operations.md)
- Fill a chest with random loot: [07 — Loot and world items](07-Loot-And-Pickups.md)
