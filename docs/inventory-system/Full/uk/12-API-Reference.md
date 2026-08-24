# 12 — Довідник API

*[🇬🇧 English](../en/12-API-Reference.md) | 🇺🇦 Українська*

Повний перелік публічного API. Розгорнуті пояснення з прикладами — у підказках
(tooltips) прямо в редакторі: наведіть курсор на будь-яке поле чи вузол.

**Усе перелічене тут викликається і з Blueprint, і з C++.** Плагін не має API,
доступного лише з коду.

---

## `UISInventoryBlueprintLibrary` — скорочення

Найкоротший шлях до типових дій. Ці функції самі знаходять потрібний компонент, тож
шукати його вручну не треба.

| Функція | Опис |
|---|---|
| `GetInventoryFor(Actor)` | знайти інвентар — на акторі, його `PlayerState` чи пешці |
| `GetEquipmentFor(Actor)` | те саме для спорядження |
| `HasInventory(Actor)` | чи є інвентар узагалі |
| `GiveItemTo(Actor, Def, Count)` | видати; повертає, **скільки реально влізло** |
| `TakeItemFrom(Actor, Def, Count, bAllowPartial)` | забрати; типово «все або нічого» |
| `GetItemCountFor(Actor, Def)` | скільки має |
| `HasItemCount(Actor, Def, Count)` | чи вистачає |
| `MoveItemBetweenActors(From, To, Slot)` | перекласти **у правильному напрямку** |
| `GetItemsWithTag(Actor, Tag)` | усі предмети з тегом |
| `GetCarriedWeight(Actor)` | сумарна вага |
| `GetItemDisplayText(Instance)` | «Меч» або «Стріла x12» |
| `GetUseResultText(Result)` | текст результату використання |

---

## `UISInventoryComponent`

### Налаштування

| Поле | Опис |
|---|---|
| `MaxSlots` | кількість слотів; **0 = необмежено** |
| `MaxWeight` | ліміт ваги; **0 = вага не враховується** |
| `AllowedItemTags` | якщо задано — приймаються лише предмети з одним із цих тегів |
| `BlockedItemTags` | предмети з цими тегами відхиляються (перевіряється **першим**) |
| `SlotRestrictions` | правила для діапазонів слотів |
| `StartingItems` | предмети, що видаються при старті (на сервері) |

### Дії — викликаються звідусіль

| Метод | Опис |
|---|---|
| `TryAddItem(Def, Count, OutRemainder)` | додати копії; `OutRemainder` — скільки не влізло |
| `TryAddItemInstance(Instance, PreferredSlot)` | додати наявний об'єкт зі збереженням стану *(сервер)* |
| `TryRemoveItemFromSlot(Slot, Count)` | вилучити з конкретного слота |
| `TryRemoveItem(Def, Count, OutRemainder)` | вилучити за типом, охоплюючи кілька стеків |
| `TryTransferTo(Target, FromSlot, ToSlot, Count)` | віддати в інший інвентар |
| `TryTransferFrom(Source, FromSlot, ToSlot, Count)` | забрати з іншого інвентаря |
| `TryQuickMoveTo(Target, FromSlot)` | «shift-click»: ціль сама обирає місце |
| `TrySplitStack(Slot, SplitCount)` | поділити стек |
| `TrySwapSlots(A, B)` | поміняти місцями або об'єднати |
| `TryMoveToSlot(From, To)` | те саме, іншими словами |
| `TryUseItem(Slot, Instigator)` | використати; повертає `EISItemUseResult` |
| `TryClearInventory()` | спорожнити |
| `SortInventory(Mode)` | впорядкувати й ущільнити від слота 0 |
| `CompactStacks()` | об'єднати неповні стеки, не переставляючи решту |

### Запити — безпечні на клієнті

| Метод | Опис |
|---|---|
| `GetItemAtSlot(Slot)` | предмет у слоті або `null` |
| `GetItemCount(Def)` | скільки копій усього |
| `HasItem(Def, Count)` | чи є принаймні стільки |
| `GetAllItems()` | усі предмети |
| `GetAllEntries()` | усі **зайняті** слоти разом з індексами |
| `GetEntriesView()` | те саме, без копії масиву *(лише C++)* |
| `FindItemSlot(Def)` | перший слот із таким типом |
| `FindAllItemSlots(Def)` | усі слоти з таким типом |
| `FindSlotOfInstance(Instance)` | де саме лежить **цей** об'єкт |
| `GetTotalItemCount()` | сума всіх стеків |
| `GetOccupiedSlotCount()` / `GetFreeSlotCount()` | зайнято / вільно |
| `GetCurrentWeight()` | поточна вага |
| `IsFull()` / `IsEmpty()` | стан |
| `IsValidSlot(Slot)` / `IsSlotOccupied(Slot)` | перевірки слота |
| `CanAcceptItem(Def)` | чи приймається такий тип узагалі |
| `CanAcceptItemCount(Def, Desired)` | **скільки реально влізе зараз** |
| `CanSlotAcceptItem(Slot, Def)` | чи прийме конкретний слот |
| `HasContainerAuthority()` | чи має ця сторона владу (спільний для інвентаря й спорядження) |
| `CanAcceptRequestFrom(Requester)` | **чи дозволено цьому акторові змінювати контейнер** |

### Збереження

`ExportState()` → `FISInventorySaveData` · `ImportState(State)` *(сервер)*

### Події

| Подія | Коли |
|---|---|
| `OnInventoryChanged` | будь-яка зміна (з типом зміни) |
| `OnItemAdded` | слот отримав предмет |
| `OnItemRemoved` | слот спорожнів |
| `OnItemChanged` | предмет той самий, дані змінилися |
| `OnAddRejected` | додавання відхилено, **з текстом причини** |

---

## `UISEquipmentComponent`

### Налаштування

| Поле | Опис |
|---|---|
| `AvailableSlots` | схема тіла — які слоти взагалі існують |
| `LinkedInventory` | рюкзак, з якого беруть і куди повертають |
| `bAutoLinkInventoryOnSameActor` | знайти інвентар на тому ж акторі (типово увімкнено) |

### Дії

| Метод | Опис |
|---|---|
| `TryEquipFromInventorySlot(Index)` | вдягнути те, що в слоті інвентаря |
| `TryEquipItem(Item, Slot, bRemoveFromInventory)` | вдягнути конкретний предмет |
| `TryUnequipSlot(Slot, bReturnToInventory)` | зняти |
| `TrySwapSlots(A, B)` | поміняти місцями *(потребує `AlternativeSlots` у предметів)* |
| `TryUnequipAll(bReturnToInventory)` | зняти все |
| `TryUseEquippedItem(Slot, Instigator)` | використати вдягнене |

### Запити

| Метод | Опис |
|---|---|
| `GetEquippedItem(Slot)` | предмет у слоті або `null` |
| `GetEquippedActor(Slot)` | візуальний актор слота |
| `IsSlotEquipped(Slot)` / `HasSlot(Slot)` | стан і наявність слота |
| `GetAllEquippedItems()` / `GetOccupiedSlots()` / `GetAllEntries()` | обхід |
| `GetEntriesView()` | обхід без копії масиву *(лише C++)* |
| `CanEquipToSlot(Item, Slot)` | чи можна вдягнути сюди |
| `GetDesiredSlotForItem(Item)` | куди предмет проситься |
| `GetTotalStatValue(StatTag)` | **сумарна статистика по всьому вдягненому** |
| `GetLinkedInventory()` | зв'язаний рюкзак (розв'язує зв'язок лінивo) |
| `HasContainerAuthority()` | чи має ця сторона владу (спільний для інвентаря й спорядження) |

### Збереження та події

`ExportState()` / `ImportState(State)` · `OnItemEquipped`, `OnItemUnequipped`,
`OnEquipmentActorSpawned`

---

## `UISItemDefinition` — тип предмета

| Поле / метод | Опис |
|---|---|
| `DisplayName`, `Description`, `Icon`, `PickupMesh` | подання |
| `ItemTags` | класифікація |
| `Fragments` | здатності |
| `FindFragmentByClass` / `HasFragment` / `GetAllFragments` | доступ до фрагментів |
| `HasAnyTags` / `HasAllTags` | перевірка тегів |
| `GetMaxStackSize()` | розмір стеку (ланцюжком крізь фрагменти) |
| `GetUnitWeight()` | вага одиниці |
| `IsUsable()` | чи має сенс «використати» |

---

## `UISItemInstance` — конкретна копія

| Поле / метод | Опис |
|---|---|
| `Definition`, `StackCount`, `StatValues` | стан |
| `GetStatValue` / `SetStatValue` / `ModifyStatValue` | статистики |
| `HasStat` / `ClearStatValue` | наявність і видалення |
| `CanStackWith(Other)` | чи можна об'єднати |
| `GetMaxStackSize` / `IsStackFull` / `GetAvailableStackSpace` | стек |
| `GetDisplayName` / `GetDescription` | подання |
| `GetStackWeight()` / `IsUsable()` | похідні |
| `DuplicateInstance(Outer)` | копія зі збереженням стану |
| `GetOwningContainer()` | **хто тримає — інвентар або спорядження** |
| `GetOwningInventory()` | інвентар, якщо це він; `null`, коли предмет вдягнений |
| `ConsumeFromContainer(Count)` | попросити того, хто тримає, витратити копії |

---

## Об'єкти світу (`InventorySystemWorld`)

### `AISLootContainer`

| Член | Опис |
|---|---|
| `ContainerInventory` | вміст |
| `LootTable`, `bGenerateLootOnSpawn` | наповнення |
| `bUseWeightedLoot`, `WeightedDropCount` | режим «рівно N предметів» |
| `bSingleUse` | одноразова скриня |
| `Open(Opener)` / `Close(Closer)` | взаємодія |
| `CanOpen()` / `IsEmpty()` | стан |
| `ResetContainer(bClear)` | дозволити наповнитися знову |
| `BP_OnOpened` / `BP_OnClosed` / `BP_OnLooted` / `BP_OnLootGenerated` | хуки візуалу |

### `AISItemPickup`

| Член | Опис |
|---|---|
| `ItemDefinition`, `ItemCount` | вміст для розміщених у рівні |
| `HeldItem` | справжній об'єкт зі станом |
| `bDestroyOnCollected`, `LifeSpanSeconds` | поведінка |
| `TryCollect(Collector)` | підібрати |
| `HasContents()` / `GetPickupText()` | стан і підпис |
| `SpawnForItem(World, Item, Location, Class)` | викинути наявний предмет у світ |

### `UISLootTable`

| Метод | Опис |
|---|---|
| `GenerateLoot()` | кидок кожного запису окремо за `DropChance` |
| `GenerateWeightedLoot(Count)` | рівно N записів за `Weight` |
| `PopulateInventory(Inv, bWeighted, Count)` | згенерувати й одразу покласти |

---

## `UISInventoryWorldSubsystem` — стан рівня світу

Усе, що спільне для гри, але **не** для процесу: підписки та таймери, які мають
зникати разом зі світом. Отримати — `UISInventoryWorldSubsystem::Get(WorldContext)`.

| Член | Опис |
|---|---|
| `GetPlayerInventory(PlayerState)` | знайти інвентар гравця, хоч на `PlayerState`, хоч на пішаку |
| `GetPlayerEquipment(PlayerState)` | те саме для спорядження |
| `OnConsumableUsed` | **подія використання витратного предмета** — предмет, хто використав, `EffectTags` |
| `GetFragmentCooldownRemaining(Fragment)` | скільки секунд лишилося на спільному кулдауні фрагмента |
| `StartFragmentCooldown(Fragment, Seconds)` | запустити його |

> Чому не статик на класі фрагмента: фрагмент спільний для всієї гри, а статик на
> ньому — ще й для всіх світів у процесі. У PIE серверний і клієнтський світи чули б
> використання один одного, а підписки з попередньої сесії лишалися б живими.

Кулдаун ключується **фрагментом**, тож він один на всіх гравців світу. Це свідомо
простий інструмент — кулдауни на гравця будуйте поверх `OnConsumableUsed`.

---

## Відлагодження (`InventorySystemDebug`)

| Функція | Опис |
|---|---|
| `ToggleInventoryDebugOverlay(PC)` | відкрити / закрити інспектор |
| `IsInventoryDebugOverlayOpen(PC)` | чи відкритий зараз |

Обидві не роблять нічого у Shipping. Консольні команди —
[13 — Відлагодження](13-Debugging.md).

---

## Перелічення

| Тип | Значення |
|---|---|
| `EISItemUseResult` | `Success`, `NoItem`, `NotUsable`, `Blocked` |
| `EISInventoryChangeType` | `ItemAdded`, `ItemRemoved`, `ItemChanged`, `SlotSwapped` |
| `EISSortMode` | `ByName`, `ByStackCount`, `ByType`, `ByWeight` |
| `EISInventoryHost` | `PlayerState`, `Pawn` |

## Структури

| Тип | Опис |
|---|---|
| `FISItemStatEntry` | одне значення в екземплярі: тег + число |
| `FISSlotRestriction` | правило для діапазону слотів |
| `FISInventoryEntry` | зайнятий слот: індекс + предмет |
| `FISEquipmentEntry` | зайнятий слот спорядження: тег + предмет + актор |
| `FISLootDrop` | результат кидка: предмет + точна кількість |
| `FISItemSaveData` / `FISInventorySaveData` / `FISEquipmentSaveData` | збереження |
