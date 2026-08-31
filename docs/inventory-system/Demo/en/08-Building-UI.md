# 08 — Building the UI

*🇬🇧 English | [🇺🇦 Українська](../uk/08-Building-UI.md)*

The plugin deliberately has no UI of its own: no two games draw an inventory the
same way. Instead it gives exactly what your interface needs — and this chapter
shows how to use that.

There's no ready widget here to "plug in". There's a list of questions everyone
who draws an inventory runs into, and answers to them.

---

## Drawing the grid

The main thing to know: **the slot list is sparse**. A 30-slot inventory with
three items stores three entries, and their indices might be 0, 7 and 29.

So there are two ways, and they're for different tasks.

### Way 1 — iterate what's there

```cpp
for (const FISDemoInventoryEntry& Entry : Inventory->GetAllEntries())
{
    DrawItem(Entry.SlotIndex, Entry.Instance);   // take the index from the entry
}
```

Good for lists: a log, a trade window, a loot panel.

### Way 2 — draw every cell

A grid must show **empty** slots too, so iterate the range, not the entries:

```cpp
const int32 SlotCount = Inventory->MaxSlots > 0 ? Inventory->MaxSlots : Inventory->GetOccupiedSlotCount();

for (int32 Slot = 0; Slot < SlotCount; ++Slot)
{
    UISDemoItemInstance* Item = Inventory->GetItemAtSlot(Slot);   // may be null
    DrawCell(Slot, Item);
}
```

> **Don't do this:** `GetAllEntries()[Index]` — entry #N is not slot #N. This is
> the most common mistake on a first attempt at drawing a grid.

In Blueprint: the **Get All Entries** node returns an array of structs, each with
`SlotIndex` and `Instance`.

---

## Updating without polling

Don't redraw the inventory every frame. Subscribe to events:

```cpp
void UMyInventoryWidget::NativeConstruct()
{
    Super::NativeConstruct();

    if (UISDemoInventoryComponent* Inv = UISDemoInventoryBlueprintLibrary::GetInventoryFor(GetOwningPlayer()))
    {
        Inv->OnInventoryChanged.AddDynamic(this, &UMyInventoryWidget::HandleChanged);
        Inv->OnAddRejected.AddDynamic(this, &UMyInventoryWidget::HandleRejected);
    }
}
```

| Event | What to redraw |
|---|---|
| `OnItemAdded` | one cell + counters |
| `OnItemRemoved` | one cell + counters |
| `OnItemChanged` | only that cell's stack label / durability bar |
| `OnInventoryChanged` | everything at once, if you don't want to distinguish |
| `OnAddRejected` | show the reason to the player |

Events fire **on both the server and the clients**, so the same widget works in
both modes.

> **Don't build the UI on the `Try*` return value.** On a client it means
> "request sent", not "succeeded" — see [09 — Multiplayer](09-Multiplayer.md).

### About `OnAddRejected`

Bind it right away. Without it the player picks up an item, nothing happens — and
they don't understand why. The event carries **ready reason text**: "Inventory
full", "Too heavy", or whatever your own fragment returned.

```cpp
void UMyInventoryWidget::HandleRejected(UISDemoItemDefinition* Def, const FText& Reason)
{
    ShowToast(Reason);
}
```

---

## The item icon

`Icon` is a **soft reference**: owning an item doesn't pull the texture into
memory. So it must be loaded before display.

**Blueprint:** the `Async Load Asset` node → result into `Set Brush from
Texture`.

**C++:**

```cpp
const TSoftObjectPtr<UTexture2D>& Icon = Instance->Definition->Icon;

if (UTexture2D* Loaded = Icon.Get())            // already in memory
{
    IconImage->SetBrushFromTexture(Loaded);
}
else if (!Icon.IsNull())
{
    UAssetManager::GetStreamableManager().RequestAsyncLoad(
        Icon.ToSoftObjectPath(),
        FStreamableDelegate::CreateWeakLambda(this, [this, Icon]()
        {
            if (UTexture2D* Now = Icon.Get())
            {
                IconImage->SetBrushFromTexture(Now);
            }
        }));
}
```

A synchronous `LoadSynchronous()` also works and is acceptable for a small
inventory, but on opening a large grid it will produce a visible hitch.

---

## The count label

Show the number only when there's more than one copy — "Sword x1" reads worse
than "Sword". A ready function already does this correctly:

```cpp
const FText Label = UISDemoInventoryBlueprintLibrary::GetItemDisplayText(Instance);
// "Sword"  or  "Arrow x12"
```

---

## The item tooltip

What's worth showing and where to get it:

| Tooltip line | Where from |
|---|---|
| Name | `Instance->GetDisplayName()` |
| Description | `Instance->GetDescription()` |
| Rarity / type | `Definition->ItemTags` (for the border colour and label) |
| Count / max | `StackCount` and `GetMaxStackSize()` |
| Stack weight | `GetStackWeight()` |
| Durability | see below |
| What it grants when worn | `Equippable.GrantedStats` |

### Durability bar

```cpp
if (const UISDemoFragment_Durability* Dur = Instance->Definition->FindFragment<UISDemoFragment_Durability>())
{
    const float Percent = Dur->GetDurabilityPercent(Instance);   // 0..1
    DurabilityBar->SetPercent(Percent);
    DurabilityBar->SetVisibility(ESlateVisibility::Visible);

    if (Dur->IsBroken(Instance))
    {
        NameText->SetColorAndOpacity(BrokenColor);
    }
}
else
{
    DurabilityBar->SetVisibility(ESlateVisibility::Collapsed);
}
```

The same approach works for any fragment: ask the item whether it has the
fragment you need, and show the corresponding line only when it does.

### Charges

```cpp
if (const UISDemoFragment_Consumable* Cons = Instance->Definition->FindFragment<UISDemoFragment_Consumable>())
{
    if (Cons->MaxCharges > 0)
    {
        ShowCharges(Cons->GetRemainingCharges(Instance), Cons->MaxCharges);
    }
}
```

---

## Drag-and-drop

### What to call on drop

```cpp
// Within one inventory
Inventory->TrySwapSlots(FromSlot, ToSlot);

// Into another inventory (chest, trade)
PlayerInventory->TryTransferTo(OtherInventory, FromSlot, ToSlot);
```

`TrySwapSlots` decides for itself what to do: swap contents, merge stacks, or
just move if the target is empty. No separate calls for those cases.

### Highlighting the target on hover

Before the drop it's worth showing whether the item will be accepted:

```cpp
const bool bWillAccept = Inventory->CanSlotAcceptItem(HoveredSlot, DraggedItem->Definition);

DropIndicator->SetColorAndOpacity(bWillAccept ? ValidColor : InvalidColor);
```

This check accounts for both the inventory-wide filters and a specific slot's
restrictions (an ammo belt). It's exactly what's missing when a player doesn't
understand why an item "won't drop".

### Splitting a stack

The classic "drag with Shift":

```cpp
Inventory->TrySplitStack(SlotIndex, Half);
```

---

## The encumbrance bar

```cpp
const float Current = Inventory->GetCurrentWeight();
const float Max     = Inventory->MaxWeight;

if (Max > 0.0f)
{
    WeightBar->SetPercent(Current / Max);
    WeightText->SetText(FText::Format(
        NSLOCTEXT("UI", "Weight", "{0} / {1}"),
        FText::AsNumber(Current), FText::AsNumber(Max)));
}
else
{
    WeightPanel->SetVisibility(ESlateVisibility::Collapsed);   // weight not limited
}
```

Weight is worth showing even when `MaxWeight = 0`, if the items have it — it's
useful information in its own right.

---

## Action buttons

Enable them by the item's state, not blindly:

```cpp
UseButton->SetIsEnabled(Instance->IsUsable());
EquipButton->SetIsEnabled(Equipment && Equipment->GetDesiredSlotForItem(Instance).IsValid());
```

`IsUsable()` says whether "use" makes sense for this item at all. Without this
check the player is offered an action that silently does nothing.

The use result is worth showing:

```cpp
const EISDemoItemUseResult Result = Inventory->TryUseItem(SlotIndex, GetOwningPlayerPawn());

if (Result != EISDemoItemUseResult::Success)
{
    ShowToast(UISDemoInventoryBlueprintLibrary::GetUseResultText(Result));
}
```

---

## The equipment panel

Iterate **`AvailableSlots`**, not the occupied slots — an empty slot needs
drawing too:

```cpp
for (const FGameplayTag& Slot : Equipment->AvailableSlots)
{
    UISDemoItemInstance* Item = Equipment->GetEquippedItem(Slot);   // may be null
    DrawEquipmentSlot(Slot, Item);
}
```

Aggregate stats for the character sheet:

```cpp
const float Armor = Equipment->GetTotalStatValue(MyTags::Stat_Armor);
```

---

## The chest window

Two grids side by side: the player's inventory and the container's contents.

```cpp
UISDemoInventoryComponent* Mine  = UISDemoInventoryBlueprintLibrary::GetInventoryFor(GetOwningPlayer());
UISDemoInventoryComponent* Chest = Container->GetContainerInventory();
```

Both are drawn the same way. The only difference is **how to move**:

```cpp
Mine->TryTransferFrom(Chest, SlotIndex);   // take
Mine->TryTransferTo(Chest, SlotIndex);     // put
Mine->TryQuickMoveTo(Chest, SlotIndex);    // double-click / shift-click
```

All three are called **on the player's inventory** — otherwise nothing happens
in multiplayer
([09 — Multiplayer](09-Multiplayer.md#the-exception-ownerless-containers)).

Don't forget to subscribe to the events of **both** inventories, so both grids
update.

---

## Where to next

- Ready scenarios: trading, crafting, a hotbar — [11 — Recipes](11-Recipes.md)
- See the state through the plugin's eyes: [13 — Debugging](13-Debugging.md)
- The full list of methods and events: [12 — API Reference](12-API-Reference.md)
