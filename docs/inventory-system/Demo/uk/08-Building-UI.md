# 08 — Побудова інтерфейсу

*[🇬🇧 English](../en/08-Building-UI.md) | 🇺🇦 Українська*

Плагін навмисно не має власного UI: жодні дві гри не малюють інвентар однаково.
Натомість він дає рівно те, що потрібно вашому інтерфейсу, — і цей розділ показує, як
цим скористатися.

Тут немає готового віджета, який треба «підключити». Є перелік питань, які виникають
у кожного, хто малює інвентар, і відповіді на них.

---

## Малювання сітки

Головне, що треба знати: **список слотів розріджений**. Інвентар на 30 слотів із
трьома предметами зберігає три записи, і їхні індекси можуть бути 0, 7 і 29.

Тому є два способи, і вони для різних задач.

### Спосіб 1 — обійти те, що є

```cpp
for (const FISDemoInventoryEntry& Entry : Inventory->GetAllEntries())
{
    DrawItem(Entry.SlotIndex, Entry.Instance);   // індекс беремо із запису
}
```

Підходить для списків: журнал, вікно обміну, панель здобичі.

### Спосіб 2 — намалювати всі клітинки

Сітка має показувати й **порожні** слоти, тож обходьте діапазон, а не записи:

```cpp
const int32 SlotCount = Inventory->MaxSlots > 0 ? Inventory->MaxSlots : Inventory->GetOccupiedSlotCount();

for (int32 Slot = 0; Slot < SlotCount; ++Slot)
{
    UISDemoItemInstance* Item = Inventory->GetItemAtSlot(Slot);   // може бути null
    DrawCell(Slot, Item);
}
```

> **Не робіть так:** `GetAllEntries()[Index]` — запис №N це не слот №N. Це найчастіша
> помилка при першій спробі намалювати сітку.

У Blueprint: вузол **Get All Entries** повертає масив структур, у кожній є
`SlotIndex` та `Instance`.

---

## Оновлення без опитування

Не перемальовуйте інвентар щокадру. Підпишіться на події:

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

| Подія | Що перемалювати |
|---|---|
| `OnItemAdded` | одну клітинку + лічильники |
| `OnItemRemoved` | одну клітинку + лічильники |
| `OnItemChanged` | лише підпис стеку / смужку міцності цієї клітинки |
| `OnInventoryChanged` | усе одразу, якщо не хочете розрізняти |
| `OnAddRejected` | показати причину гравцеві |

Події спрацьовують **і на сервері, і на клієнтах**, тож той самий віджет працює в
обох режимах.

> **Не будуйте UI на поверненому значенні `Try*`.** На клієнті воно означає «запит
> надіслано», а не «вдалося» — див. [09 — Мультиплеєр](09-Multiplayer.md).

### Про `OnAddRejected`

Прив'яжіть її одразу. Без неї гравець підбирає предмет, нічого не відбувається — і він
не розуміє чому. Подія несе **готовий текст причини**: «Інвентар повний», «Надто
важко», або те, що повернув ваш власний фрагмент.

```cpp
void UMyInventoryWidget::HandleRejected(UISDemoItemDefinition* Def, const FText& Reason)
{
    ShowToast(Reason);
}
```

---

## Іконка предмета

`Icon` — **м'яке посилання**: володіння предметом не тягне текстуру в пам'ять. Тому
перед показом її треба завантажити.

**Blueprint:** вузол `Async Load Asset` → результат у `Set Brush from Texture`.

**C++:**

```cpp
const TSoftObjectPtr<UTexture2D>& Icon = Instance->Definition->Icon;

if (UTexture2D* Loaded = Icon.Get())            // вже в пам'яті
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

Синхронний `LoadSynchronous()` теж працює і прийнятний для невеликого інвентаря, але
на відкритті великої сітки дасть помітний ривок.

---

## Підпис кількості

Показуйте число лише коли копій більше однієї — «Меч x1» читається гірше, ніж «Меч».
Готова функція вже робить це правильно:

```cpp
const FText Label = UISDemoInventoryBlueprintLibrary::GetItemDisplayText(Instance);
// «Меч»  або  «Стріла x12»
```

---

## Тултип предмета

Що варто показати й звідки це взяти:

| Рядок тултипа | Звідки |
|---|---|
| Назва | `Instance->GetDisplayName()` |
| Опис | `Instance->GetDescription()` |
| Рідкість / тип | `Definition->ItemTags` (для кольору рамки й підпису) |
| Кількість / максимум | `StackCount` та `GetMaxStackSize()` |
| Вага стеку | `GetStackWeight()` |
| Міцність | див. нижче |
| Що дає при вдяганні | `Equippable.GrantedStats` |

### Смужка міцності

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

Той самий підхід працює для будь-якого фрагмента: спитайте предмет, чи є в нього
потрібний фрагмент, і покажіть відповідний рядок лише тоді, коли він є.

### Заряди

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

## Перетягування

### Що викликати при відпусканні

```cpp
// Усередині одного інвентаря
Inventory->TrySwapSlots(FromSlot, ToSlot);

// В інший інвентар (скриня, обмін)
PlayerInventory->TryTransferTo(OtherInventory, FromSlot, ToSlot);
```

`TrySwapSlots` сам вирішує, що робити: обміняти вміст, об'єднати стеки, або просто
перемістити, якщо ціль порожня. Окремих викликів для цих випадків не потрібно.

### Підсвітка цілі при наведенні

Перед відпусканням варто показати, чи приймуть предмет:

```cpp
const bool bWillAccept = Inventory->CanSlotAcceptItem(HoveredSlot, DraggedItem->Definition);

DropIndicator->SetColorAndOpacity(bWillAccept ? ValidColor : InvalidColor);
```

Ця перевірка врахує і загальні фільтри інвентаря, і обмеження конкретного слота
(патронний пояс). Саме її бракує, коли гравець не розуміє, чому предмет «не лягає».

### Поділ стеку

Класичне «перетягнути з Shift»:

```cpp
Inventory->TrySplitStack(SlotIndex, Half);
```

---

## Смужка навантаження

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
    WeightPanel->SetVisibility(ESlateVisibility::Collapsed);   // вага не обмежена
}
```

Вагу варто показувати навіть тоді, коли `MaxWeight = 0`, якщо предмети її мають — це
корисна інформація сама по собі.

---

## Кнопки дій

Вмикайте їх за станом предмета, а не наосліп:

```cpp
UseButton->SetIsEnabled(Instance->IsUsable());
EquipButton->SetIsEnabled(Equipment && Equipment->GetDesiredSlotForItem(Instance).IsValid());
```

`IsUsable()` каже, чи взагалі має сенс «використати» цей предмет. Без цієї перевірки
гравцеві пропонують дію, яка мовчки нічого не робить.

Результат використання варто показати:

```cpp
const EISDemoItemUseResult Result = Inventory->TryUseItem(SlotIndex, GetOwningPlayerPawn());

if (Result != EISDemoItemUseResult::Success)
{
    ShowToast(UISDemoInventoryBlueprintLibrary::GetUseResultText(Result));
}
```

---

## Панель спорядження

Обходьте **`AvailableSlots`**, а не зайняті слоти — порожній слот теж треба намалювати:

```cpp
for (const FGameplayTag& Slot : Equipment->AvailableSlots)
{
    UISDemoItemInstance* Item = Equipment->GetEquippedItem(Slot);   // може бути null
    DrawEquipmentSlot(Slot, Item);
}
```

Сумарні статистики для аркуша персонажа:

```cpp
const float Armor = Equipment->GetTotalStatValue(MyTags::Stat_Armor);
```

---

## Вікно скрині

Дві сітки поруч: інвентар гравця й вміст контейнера.

```cpp
UISDemoInventoryComponent* Mine  = UISDemoInventoryBlueprintLibrary::GetInventoryFor(GetOwningPlayer());
UISDemoInventoryComponent* Chest = Container->GetContainerInventory();
```

Обидві малюються однаково. Різниця лише в тому, **як переміщати**:

```cpp
Mine->TryTransferFrom(Chest, SlotIndex);   // забрати
Mine->TryTransferTo(Chest, SlotIndex);     // покласти
Mine->TryQuickMoveTo(Chest, SlotIndex);    // подвійний клік / shift-клік
```

Усі три викликаються **на інвентарі гравця** — інакше в мультиплеєрі нічого не
станеться ([09 — Мультиплеєр](09-Multiplayer.md#виняток-контейнери-без-власника)).

Не забудьте підписатися на події **обох** інвентарів, щоб обидві сітки оновлювалися.

---

## Куди далі

- Готові сценарії: торгівля, крафт, хотбар — [11 — Рецепти](11-Recipes.md)
- Побачити стан очима плагіна: [13 — Відлагодження](13-Debugging.md)
- Повний перелік методів і подій: [12 — Довідник API](12-API-Reference.md)
