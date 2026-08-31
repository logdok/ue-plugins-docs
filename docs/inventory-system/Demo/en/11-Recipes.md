# 11 — Recipes

*🇬🇧 English | [🇺🇦 Українська](../uk/11-Recipes.md)*

Ready solutions to common tasks. Each is self-contained: copy the closest one
and tweak it.

Everything shown works in single-player and multiplayer, unchanged.

---

## Crafting

Spend materials and grant the result. The key thing here is **not to spend half
of it**.

```cpp
bool AMyCraftingStation::TryCraft(AActor* Player, const FMyRecipe& Recipe)
{
    UISDemoInventoryComponent* Inv = UISDemoInventoryBlueprintLibrary::GetInventoryFor(Player);
    if (!Inv)
    {
        return false;
    }

    // 1. Check EVERYTHING before spending anything.
    for (const TPair<UISDemoItemDefinition*, int32>& Ingredient : Recipe.Ingredients)
    {
        if (!Inv->HasItem(Ingredient.Key, Ingredient.Value))
        {
            return false;   // missing - touch nothing
        }
    }

    // 2. Check that the result will have somewhere to go.
    if (Inv->CanAcceptItemCount(Recipe.Result, Recipe.ResultCount) < Recipe.ResultCount)
    {
        return false;   // otherwise the materials would vanish, but the sword wouldn't
    }

    // 3. Now spending is safe.
    for (const TPair<UISDemoItemDefinition*, int32>& Ingredient : Recipe.Ingredients)
    {
        int32 Remainder = 0;
        Inv->TryRemoveItem(Ingredient.Key, Ingredient.Value, Remainder);
    }

    int32 Remainder = 0;
    Inv->TryAddItem(Recipe.Result, Recipe.ResultCount, Remainder);
    return true;
}
```

The two check steps aren't excessive caution. Without the first, the player
loses some materials on a failed craft; without the second, they lose all of
them, and the result doesn't fit.

---

## Trading

A purchase is a currency check, a swap, and no intermediate state where the
player has already paid but not yet received.

```cpp
bool AMyVendor::TryBuy(AActor* Buyer, int32 VendorSlot, int32 Count)
{
    UISDemoInventoryComponent* BuyerInv = UISDemoInventoryBlueprintLibrary::GetInventoryFor(Buyer);
    if (!BuyerInv || !Stock)
    {
        return false;
    }

    UISDemoItemInstance* Goods = Stock->GetItemAtSlot(VendorSlot);
    if (!Goods)
    {
        return false;
    }

    const int32 Price = GetPrice(Goods->Definition) * Count;

    // Money? Room?
    if (!BuyerInv->HasItem(CurrencyDefinition, Price))
    {
        return false;
    }
    if (BuyerInv->CanAcceptItemCount(Goods->Definition, Count) < Count)
    {
        return false;
    }

    // Spend the currency first, then hand over the goods.
    int32 Remainder = 0;
    BuyerInv->TryRemoveItem(CurrencyDefinition, Price, Remainder);
    BuyerInv->TryTransferFrom(Stock, VendorSlot, -1, Count);

    return true;
}
```

**Selling** is the mirror image: `BuyerInv->TryTransferTo(Stock, PlayerSlot)`
and grant currency.

To stop the shop accepting quest items, configure filters on its inventory:

```
Stock->AllowedItemTags: Item.Type.Weapon, Item.Type.Armor
Stock->BlockedItemTags: Item.Property.QuestItem
```

Then `TryTransferTo` refuses the unsuitable ones itself, with no check in your
code.

---

## Currency as an item

The simplest way — an ordinary item with a big stack:

```
DA_Gold
├── ItemTags: Item.Type.Currency
└── Fragments: Stackable → MaxStackSize: 99999
```

Then money lives in the inventory, is saved along with it, and is shown by the
same UI.

```cpp
const int32 Gold = Inventory->GetItemCount(GoldDefinition);
```

If currency shouldn't take a slot, keep it outside the inventory, in your own
component. The plugin doesn't insist otherwise.

---

## A hotbar (quick-access bar)

The simplest solution: the hotbar is the **first N slots** of the same
inventory.

```cpp
// The player pressed "3"
Inventory->TryUseItem(2, GetPawn());   // slots 0..N-1 = buttons 1..N
```

Nothing to synchronise: there's one slot, the UI shows it twice.

If the hotbar should accept only certain things — add a slot restriction:

```
SlotRestrictions:
  └── [0]: FirstSlot 0, LastSlot 4, RequiredTags: Item.Type.Consumable
```

A more complex variant — the hotbar as separate **references** to slots of the
main inventory (an array of indices in your UI). Then the item stays put and the
hotbar just points at it. The plugin doesn't get in the way — that's purely your
UI state.

---

## Looting a corpse

A corpse is an actor with an inventory component. Nothing special.

```cpp
void AMyCharacter::OnDeath()
{
    // Don't delete the actor right away - its inventory is the loot.
    SetLifeSpan(120.0f);
    bIsLootable = true;
}
```

The player takes from it the same way as from a chest:

```cpp
PlayerInventory->TryTransferFrom(CorpseInventory, SlotIndex);
```

To instead **spill everything on the ground**:

```cpp
void AMyCharacter::DropEverything()
{
    if (!HasAuthority()) return;

    for (const FISDemoInventoryEntry& Entry : Inventory->GetAllEntries())
    {
        UISDemoItemInstance* Item = Entry.Instance;
        const int32 Slot = Entry.SlotIndex;

        // Take out of the inventory, then hand to a pickup - the item keeps its state.
        Inventory->TryRemoveItemFromSlot(Slot, Item->StackCount);
        AISDemoItemPickup::SpawnForItem(this, Item, GetActorLocation() + FMath::VRand() * 100.0f);
    }
}
```

> Iterate a **copy** of the list if you change the inventory inside the loop.
> Simpler still — collect the indices in advance, then act.

---

## Dropping an item

```cpp
void AMyCharacter::DropItem(int32 SlotIndex)
{
    UISDemoItemInstance* Item = Inventory->GetItemAtSlot(SlotIndex);
    if (!Item) return;

    const FVector DropAt = GetActorLocation() + GetActorForwardVector() * 150.0f;

    Inventory->TryRemoveItemFromSlot(SlotIndex, Item->StackCount);
    AISDemoItemPickup::SpawnForItem(this, Item, DropAt);
}
```

Order matters: first take it out of the inventory, then hand it to the pickup.
The pickup **takes ownership** of the object itself, so a sword at 12/100
durability lies on the ground exactly like that.

To forbid dropping quest items — check the tag:

```cpp
if (Item->Definition->ItemTags.HasTag(QuestItemTag))
{
    ShowToast(NSLOCTEXT("UI", "CantDrop", "This can't be dropped."));
    return;
}
```

---

## Pickup on touch

```cpp
void AMyPickupTrigger::NotifyActorBeginOverlap(AActor* Other)
{
    Super::NotifyActorBeginOverlap(Other);

    if (!HasAuthority()) return;   // the server collects

    if (UISDemoInventoryBlueprintLibrary::HasInventory(Other))
    {
        TryCollect(Other);
    }
}
```

`HasInventory` saves needless work when something without an inventory enters
the volume.

---

## A reward for a quest or event

The shortest path — no component lookup:

```cpp
const int32 Given = UISDemoInventoryBlueprintLibrary::GiveItemTo(Player, RewardDef, 3);

if (Given < 3)
{
    // Not everything fit - spill the rest at their feet.
    SpawnRemainderAsPickup(Player, RewardDef, 3 - Given);
}
```

---

## A starting kit

No code at all: fill `StartingItems` on the inventory component in the character
Blueprint. The items are granted on the server at startup.

If the kit depends on the character class — code is simpler:

```cpp
void AMyCharacter::BeginPlay()
{
    Super::BeginPlay();

    if (HasAuthority())
    {
        for (const TPair<UISDemoItemDefinition*, int32>& Entry : ClassStartingKit)
        {
            int32 Remainder = 0;
            Inventory->TryAddItem(Entry.Key, Entry.Value, Remainder);
        }
    }
}
```

---

## Repairing equipment

```cpp
void AMyBlacksmith::Repair(UISDemoItemInstance* Item)
{
    if (!Item || !Item->Definition) return;

    if (const UISDemoFragment_Durability* Dur = Item->Definition->FindFragment<UISDemoFragment_Durability>())
    {
        Dur->Repair(Item);   // a negative value = full repair
    }
}
```

The cost is convenient to compute from how much is missing:

```cpp
const float Missing = Dur->MaxDurability - Dur->GetCurrentDurability(Item);
const int32 Cost = FMath::CeilToInt(Missing * CostPerPoint);
```

---

## Extending the component

When you need custom behaviour on every inventory — subclass:

```cpp
UCLASS()
class UMyInventoryComponent : public UISDemoInventoryComponent
{
    GENERATED_BODY()

public:
    UMyInventoryComponent()
    {
        MaxSlots = 40;
        MaxWeight = 120.0f;
    }
};
```

Then either place your class in the character Blueprint, or name it in
**Project Settings → Game → Inventory System → Inventory Component Class** so
auto-creation uses it.

The same works for `UISDemoEquipmentComponent` and for any fragment.

---

## Several inventories on one actor

Just add several components — a backpack, a wallet, a quest bag:

```cpp
Backpack   = CreateDefaultSubobject<UISDemoInventoryComponent>(TEXT("Backpack"));
QuestBag   = CreateDefaultSubobject<UISDemoInventoryComponent>(TEXT("QuestBag"));

QuestBag->AllowedItemTags.AddTag(QuestItemTag);
```

Remember: `GetInventoryFor` returns the **first one found**. If there are
several, either keep the references explicitly, or implement
`IISDemoInventoryInterface` and return the one you consider primary:

```cpp
virtual UISDemoInventoryComponent* GetInventoryComponent() const override { return Backpack; }
```

The interface takes priority over the search — that's exactly the case it exists
for.

---

## Where to next

- Draw all of this: [08 — Building the UI](08-Building-UI.md)
- Write your own fragment: [04 — Fragments](04-Fragments.md#a-custom-fragment)
- Check the result: [13 — Debugging](13-Debugging.md)
