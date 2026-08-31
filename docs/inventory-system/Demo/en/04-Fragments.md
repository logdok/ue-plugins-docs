# 04 — Fragments

*🇬🇧 English | [🇺🇦 Українська](../uk/04-Fragments.md)*

A fragment is one capability of an item. An item type has no behaviour of its
own: it only lists fragments, and they do all the work.

---

## The main rule: fragments are shared

The fragment object lives **inside the asset** `ISItemDefinition`. There is
**one for the whole game**, shared by every instance of that item.

If a hundred players are holding torches, all hundred torches point at the
**same** `UISDemoFragment_Durability` object.

That's why every hook is declared `const`, and writing into a fragment field at
runtime is always a bug:

```cpp
// WRONG - changes the asset for every torch in the game at once
CurrentDurability -= 1.0f;

// RIGHT - changes only the torch that was used
Instance->ModifyStatValue(ISDemoTags::Stat_Durability, -1.0f);
```

The mutable state of a particular copy is stored in
`UISDemoItemInstance::StatValues` — a list of numeric values keyed by tags.

> **A side effect worth remembering.** An item that has even one value in
> `StatValues` never merges into a stack — see
> [01 — Core Concepts](01-Core-Concepts.md#stacking-rules).

---

## Stackable — several copies in one slot

| Field | Description |
|---|---|
| `MaxStackSize` | how many copies fit in one slot (minimum 2) |

Without this fragment an item takes a slot per copy.

Typical values: `99` for materials and currency, `20` for potions, `5–10` for
bulky things.

The fragment alone doesn't guarantee that two instances will merge into a
stack: `CanStackWith` is responsible for that, and an instance's individual
state always has the last word. If an item has its own data — `Durability`, say
— two instances with different values **will never merge**, even if both stack.
Otherwise a chipped sword could "dissolve" into a pile of new ones. So
`Stackable + Durability` together only makes sense when the item starts without
an individual stamp and picks up wear afterward.

---

## Equippable — the item can be worn

| Field | Description |
|---|---|
| `EquipmentSlot` | the default slot — where the item goes when no destination is named |
| `AlternativeSlots` | **other slots the item can also live in** — what makes two hands and `TrySwapSlots` possible |
| `EquippedActorClass` | an actor that appears when equipped (optional) |
| `AttachSocketName` | a socket on the character's mesh |
| `AttachOffset` | a transform tweak after attachment |
| `GrantedStats` | stats the item grants while worn |

The slot named here must be present in the equipment component's
`AvailableSlots` — otherwise the item has nowhere to be equipped.

The visual actor is created **locally on each machine** and is not replicated.
So a sword in a character's hand costs no traffic beyond the slot entry itself.

`GrantedStats` are declarative: the plugin only sums them via
`GetTotalStatValue`, and what "armour 12" means is decided by your game.
Details in [06 — Equipment](06-Equipment.md).

> **One item, one slot.** A ring that fits either of two slots isn't expressed
> here directly: give both slots a shared tag
> (`Equipment.Slot.Accessory.Ring`) and open several such slots on the
> character.

---

## Consumable — the item is spent

| Field | Description |
|---|---|
| `ConsumeAmount` | how many copies one use spends |
| `MaxCharges` | charges per copy (0 = copies are spent) |
| `bDestroyWhenChargesSpent` | destroy after the last charge |
| `EffectTags` | what exactly should happen — in your terms |
| `UseCooldown` | minimum seconds between uses |

### Two modes

**Stack mode** (`MaxCharges = 0`) — a potion, once drunk, disappears.

**Charges mode** (`MaxCharges > 0`) — the item stays in the slot spending
charges: a wand of 10 spells, a lantern with fuel, a medkit with 3 uses.

### The effect is yours to implement

The fragment does the bookkeeping — spends a copy or a charge — and reports
**what** was consumed, through `EffectTags`. It never applies effects itself.

This is deliberate: "restore 50 health" means something different in every
project, and a plugin that tried to guess would be wrong in most of them.

Subscribe once — in the GameMode, for example:

```cpp
void AMyGameMode::BeginPlay()
{
    Super::BeginPlay();

    // The event lives on the world subsystem, not on the fragment class: the
    // fragment is shared by the whole game, and a static on it would be shared
    // by every world in the process too. In PIE the server and client would
    // hear each other's uses.
    if (UISDemoInventoryWorldSubsystem* Subsystem = UISDemoInventoryWorldSubsystem::Get(this))
    {
        Subsystem->OnConsumableUsed.AddDynamic(this, &AMyGameMode::HandleConsumableUsed);
    }
}

void AMyGameMode::HandleConsumableUsed(
    UISDemoItemInstance* Item, AActor* Instigator, const FGameplayTagContainer& EffectTags)
{
    if (EffectTags.HasTag(MyTags::Effect_Heal))
    {
        ApplyHealing(Instigator, 50.0f);
    }
    else if (EffectTags.HasTag(MyTags::Effect_RestoreMana))
    {
        ApplyMana(Instigator, 30.0f);
    }
}
```

If you use the Gameplay Ability System, this is the natural place to apply a
`GameplayEffect` by tag.

> `UseCooldown` is a deliberately simple cooldown: **one per fragment, shared
> by every player in the world**. For single-player that's what you want; for
> a competitive game, obviously not, because two players would share one potion
> timer. Per-player cooldowns, cooldown categories and a UI countdown belong to
> your systems: leave `UseCooldown` at zero and count them yourself by
> subscribing to `OnConsumableUsed`.
>
> The cooldown itself is stored by `UISDemoInventoryWorldSubsystem` — it disappears
> with the world. That's exactly why it isn't a static field on the fragment: a
> static would survive a PIE session and mix the time of the server and client
> worlds, which in the editor live in one process.

---

## Durability — the item wears down

| Field | Description |
|---|---|
| `MaxDurability` | durability of a new copy |
| `DurabilityPerUse` | how much one use costs |
| `bDestroyWhenBroken` | destroy on reaching zero |
| `bCannotUseWhenBroken` | don't allow using a broken item |

The fragment is completely self-contained: it stamps full durability on a new
copy, deducts wear, and refuses to use a broken tool — all by itself. The
inventory core knows nothing about durability.

Useful functions: `GetDurabilityPercent` (0–1, handy for a bar), `IsBroken`,
`Repair`.

**Wear not from use.** Set `DurabilityPerUse = 0` if your game damages
equipment from elsewhere (taking a hit, environmental exposure), and call
`ModifyStatValue` directly.

---

## Weight — the item has weight

| Field | Description |
|---|---|
| `UnitWeight` | weight of one copy |

Counted only by inventories with a non-zero `MaxWeight`. Otherwise the value is
purely informational — the UI can display it.

The unit is yours: kilograms, pounds, abstract "load units". The plugin only
compares the sum against the limit.

---

## A custom fragment

### In Blueprint

1. Create a Blueprint class with **ISItemFragment** as the parent.
2. Override the events you need.
3. Add it to the `Fragments` array of any item.

No C++ required.

### In C++

```cpp
UCLASS(DisplayName = "Soulbound")
class UISFragment_Soulbound : public UISDemoItemFragment
{
    GENERATED_BODY()

public:
    // Remember the owner when a copy is created.
    virtual void OnInstanceCreated_Implementation(UISDemoItemInstance* Instance) const override
    {
        // Write state to the instance, not to the fragment's fields.
        Instance->SetStatValue(MyTags::Stat_BoundTo, 0.0f);
    }

    // Forbid the item from entering someone else's inventory.
    virtual bool CanBeAddedTo_Implementation(
        UISDemoItemInstance* Instance,
        UISDemoInventoryComponent* Inventory,
        FText& OutReason) const override
    {
        if (!BelongsTo(Instance, Inventory))
        {
            OutReason = NSLOCTEXT("MyGame", "Bound", "This item belongs to someone else.");
            return false;
        }
        return true;
    }
};
```

The `OutReason` text reaches the player through the `OnAddRejected` event —
show it as a popup.

### How to spend the item from a fragment

```cpp
Instance->ConsumeFromContainer(1);   // this way
```

This is a request to **whoever holds the item** — no matter whether it's an
inventory or an equipment slot. Don't look up the slot index yourself:

```cpp
// NO - a worn item has no slot index, and this call silently does nothing
UISDemoInventoryComponent* Inv = Instance->GetOwningInventory();
Inv->TryRemoveItemFromSlot(Inv->FindSlotOfInstance(Instance), 1);
```

This is exactly what the built-in `Consumable` once tripped over: a drunk
**worn** flask announced its effect and wasn't spent.

The same goes for writing a stat: `SetStatValue` finds whoever holds the item
itself, flags the slot for replication and raises the event — for both a
backpack and equipment.

---

## Hook reference

### Lifecycle

| Hook | When it's called |
|---|---|
| `OnInstanceCreated` | right after a new copy is created — set starting values here |
| `OnAddedToInventory` | the item entered an inventory |
| `OnRemovedFromInventory` | the item is leaving an inventory (still inside) |

`OnInstanceCreated` is **not** called on a stack split, a move or a save load —
otherwise a broken item would "heal" from every transfer.

### Vetoes — **every** fragment must agree

| Hook | Question |
|---|---|
| `CanBeAddedTo` | can the item be added to this inventory? |
| `CanStackWith` | can two stacks merge? |
| `CanBeUsed` | can the item be used right now? |

One `false` stops the action. The `OutReason` text is shown to the player.

### Actions

| Hook | When |
|---|---|
| `IsUsable` | whether "use" makes sense for this item at all |
| `OnUsed` | apply the use effect |
| `OnEquipped` / `OnUnequipped` | the item was equipped / is being unequipped |

### Value modifiers

| Hook | What it does |
|---|---|
| `ModifyMaxStackSize` | affects the stack size |
| `ModifyUnitWeight` | affects the unit weight |

These two work as a **chain**: the value passes through every fragment in turn.
That's why an item without `Stackable` has a stack of 1 — nobody changed the
initial value.

> **Determinism is mandatory.** Query hooks (`CanStackWith`,
> `ModifyMaxStackSize`, `ModifyUnitWeight`) run **on clients too**. If a client
> computes a different stack size than the server, the player sees a
> desynchronised inventory.
>
> Hooks that change state (`OnUsed`, `OnAddedToInventory`, `OnEquipped`…) run
> **on the server only**.

---

## Where to next

- See fragments in action: [05 — Inventory Operations](05-Inventory-Operations.md)
- Extend the plugin with your own code: [11 — Recipes](11-Recipes.md)
