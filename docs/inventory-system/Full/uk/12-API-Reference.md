# 12 — Довідник API

*[🇬🇧 English](../en/12-API-Reference.md) | 🇺🇦 Українська*

Повний перелік публічного API. Розгорнуті пояснення з прикладами — у підказках
(tooltips) прямо в редакторі: наведіть курсор на будь-яке поле чи вузол.

**Майже все перелічене тут викликається і з Blueprint, і з C++.** Винятки позначені
*(лише C++)* і їх лише два: `GetEntriesView` на обох контейнерах, який віддає масив
без копії, та інтерфейс `IISInventoryInterface`. Жодної **можливості** плагіна вони
не додають — обидва мають блюпринтові відповідники (`GetAllEntries` і пошук
компонента через `UISInventoryBlueprintLibrary`).

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
| `MaxSlots` | кількість слотів; **0 = необмежено**. Реплікується |
| `MaxWeight` | ліміт ваги; **0 = вага не враховується**. Реплікується |
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
| `TryCollect(Collectible)` | **підібрати предмет зі світу** — викликати на інвентарі гравця, не на пікапі |
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
| `GetLinkedInventory()` | зв'язаний рюкзак; якщо кеш порожній — шукає на акторі |
| `HasContainerAuthority()` | чи має ця сторона владу (спільний для інвентаря й спорядження) |

### Збереження та події

`ExportState()` / `ImportState(State)`

| Подія | Коли |
|---|---|
| `OnItemEquipped` | слот **став** зайнятим — саме вдягання, а не будь-яка зміна |
| `OnItemUnequipped` | слот звільнився |
| `OnEquippedItemChanged` | предмет той самий, його дані змінилися (міцність, заряди) |
| `OnEquipmentActorSpawned` | візуальний актор слота створено й прикріплено |

> Три перші події — рівно та сама трійка, що й в інвентаря
> (`OnItemAdded` / `OnItemRemoved` / `OnItemChanged`). Звук вдягання прив'язуйте до
> `OnItemEquipped`: він не спрацює вдруге від того, що меч втратив одиницю міцності.

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

## Фрагменти — власні помічники

Поля кожного фрагмента описані в [04 — Фрагменти](04-Fragments.md). Крім полів,
деякі фрагменти дають функції, які можна викликати напряму:

| Фрагмент | Функція | Опис |
|---|---|---|
| `UISFragment_Durability` | `GetCurrentDurability(Instance)` | поточна міцність копії |
| | `GetDurabilityPercent(Instance)` | 0–1, готове для прогрес-бара |
| | `IsBroken(Instance)` | чи впала міцність до нуля |
| | `Repair(Instance, Amount)` | полагодити; від'ємне значення — повністю *(сервер)* |
| `UISFragment_Consumable` | `GetRemainingCharges(Instance)` | скільки зарядів лишилося |
| `UISFragment_Equippable` | `AcceptsSlot(Slot)` | чи згоден предмет жити в цьому слоті |

Усі дванадцять хуків, які перевизначає власний фрагмент, — теж публічний API:
`OnInstanceCreated`, `OnAddedToInventory`, `OnRemovedFromInventory`, `CanBeAddedTo`,
`CanStackWith`, `CanBeUsed`, `IsUsable`, `OnUsed`, `OnEquipped`, `OnUnequipped`,
`ModifyMaxStackSize`, `ModifyUnitWeight`.

---

## `UISInventoryQueryLibrary` — пошук за описом

Коли питання складніше за «скільки в мене цього»: «уся зброя, що зношена», «усе, що
можна продати», «перший стек стріл, де більше десяти».

| Функція | Опис |
|---|---|
| `QueryInventory(Inv, Query)` | усі предмети, що підходять |
| `QueryInventoryFirst(Inv, Query)` | перший збіг або `null` |
| `QueryInventoryCount(Inv, Query)` | сумарна кількість копій у всіх збігах |
| `QueryInventorySlots(Inv, Query)` | індекси слотів усіх збігів, за зростанням |
| `MakeTagQuery(Tag)` | скорочення: усе з одним тегом |
| `MakeDefinitionQuery(Def)` | скорочення: рівно один тип предмета |

Поля `FISInventoryQuery` (порожній запит підходить усьому, поля комбінуються через І):

| Поле | Опис |
|---|---|
| `RequiredTags` | предмет має нести **всі** ці теги |
| `AnyOfTags` | предмет має нести **хоч один** із цих тегів |
| `ExcludedTags` | предмет із будь-яким із цих тегів виключається |
| `RequiredFragment` | предмет має мати фрагмент цього класу (підкласи теж) |
| `MinStackCount` / `MaxStackCount` | межі розміру стеку |
| `SpecificDefinition` | рівно цей тип предмета |

---

## Інтерфейси — точки інтеграції

| Інтерфейс | Навіщо |
|---|---|
| `IISCollectibleInterface` | зробити своїм актором те, що вміє підбирати `UISInventoryComponent::TryCollect`. `AISItemPickup` уже його реалізує; свій — лише якщо ваш «підбирний» об'єкт не є його нащадком. Метод один: `TryGiveContents(Collector)`, викликається вже на сервері |
| `IISInventoryInterface` | сказати, який із кількох інвентарів на акторі головний — `GetInventoryComponent`, `GetEquipmentComponent`, `GetAllInventoryComponents`. Потрібен рідко: `GetInventoryFor` і сам знаходить єдиний. *(лише C++)* |

---

## Об'єкти світу (`InventorySystemWorld`)

### `AISLootContainer`

| Член | Опис |
|---|---|
| `ContainerInventory` | вміст |
| `LootTable`, `bGenerateLootOnSpawn` | наповнення |
| `bUseWeightedLoot`, `WeightedDropCount` | режим «рівно N предметів» |
| `bSingleUse` | одноразова скриня |
| `Open(Opener)` / `Close(Closer)` | взаємодія — **лише на сервері**, самі себе не пересилають |
| `CanOpen()` / `IsEmpty()` | стан |
| `ResetContainer(bClear)` | дозволити наповнитися знову |
| `BP_OnOpened` / `BP_OnClosed` / `BP_OnLooted` / `BP_OnLootGenerated` | хуки візуалу |

### `AISItemPickup`

| Член | Опис |
|---|---|
| `ItemDefinition`, `ItemCount` | вміст для розміщених у рівні |
| `HeldItem` | справжній об'єкт зі станом |
| `bDestroyOnCollected`, `LifeSpanSeconds` | поведінка |
| `TryCollect(Collector)` | підібрати — **лише на сервері**; з клієнта викликайте `UISInventoryComponent::TryCollect` |
| `HasContents()` / `GetPickupText()` | стан і підпис |
| `SpawnForItem(World, Item, Location, Class)` | викинути наявний предмет у світ *(сервер)* |

> **Який `TryCollect` викликати.** Той, що на інвентарі гравця. Пікап не має
> мережевого з'єднання, тож сам переслати запит на сервер не може — а інвентар
> гравця може. Це та сама причина, з якої з контейнера беруть через
> `TryTransferFrom`, а не через методи самого контейнера.

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
> ньому був би спільним ще й для всіх світів у процесі. У PIE серверний і клієнтський
> світи чули б використання один одного, а підписки з попередньої сесії лишалися б
> живими в наступній.

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
| `FISLootTableEntry` | запис таблиці луту: предмет, діапазон кількості, шанс і вага |
| `FISLootDrop` | результат кидка: предмет + точна кількість |
| `FISInventoryQuery` | опис того, що шукаємо в інвентарі |
| `FISItemSaveData` / `FISInventorySaveData` / `FISEquipmentSaveData` | збереження |

## Налаштування проєкту

`UISInventorySettings` — **Project Settings → Game → Inventory System**. Усе
вимкнено або нейтральне за замовчуванням; докладно в
[02 — Підключення](02-Setup.md).

| Поле | Опис |
|---|---|
| `bAutoCreatePlayerInventory` | видавати інвентар кожному гравцеві автоматично |
| `bAutoCreatePlayerEquipment` | і спорядження разом із ним |
| `AutoInventoryHost` | де він живе: `PlayerState` (переживає смерть) чи `Pawn` |
| `DefaultInventorySlots` | скільки слотів отримує автоматичний інвентар |
| `DefaultEquipmentSlots` | схема тіла для автоматичного спорядження |
| `InventoryComponentClass` / `EquipmentComponentClass` | ваші підкласи замість стандартних |
| `MaxInteractionDistance` | серверна межа дальності запитів; 0 — не перевіряти |
| `bValidateItemsOnStartup` | перевіряти ассети предметів при старті редактора |
| `bVerboseLogging` | докладний лог кожної зміни інвентаря |
