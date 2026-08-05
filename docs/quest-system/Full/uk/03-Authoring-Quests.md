# 03 — Авторинг квестів

*[🇬🇧 English](../en/03-Authoring-Quests.md) | 🇺🇦 Українська*

Квести й цілі — це `UPrimaryDataAsset`. Створюйте їх у Content Browser: **Add → Miscellaneous → Data Asset**, потім оберіть клас `QuestObjectiveData` або `QuestData`.

Спочатку створіть цілі, потім пошліться на них із квесту.

## UQuestObjectiveData

| Властивість | Тип | Значення |
|----------|------|---------|
| `ObjectiveName` | `FText` | Коротка відображувана назва ("Defeat the Bandit Leader"). |
| `ObjectiveDescription` | `FText` | Довший опис для UI журналу квестів. |
| `ObjectiveType` | `EQuestObjectiveType` | Яка дія завершує цю ціль — див. нижче. |
| `TargetCount` | `int32` (≥ 1) | Скільки разів має відбутися дія. |
| `bIsOptional` | `bool` | Необов'язкові цілі не блокують завершення квесту. |
| `bIsSequential` | `bool` | Якщо true, ця ціль стає активною лише після завершення попередніх послідовних цілей у цьому ж квесті. Див. [04](04-Event-Driven-Progress.md). |
| `ObjectiveIdentifier` | `FName` | Значення, з яким мають збігтися ігрові події (наприклад, `"Bandit"`, `"HealthPotion"`). |
| `ExpectedEventID` | `FName` | Використовується лише за `ObjectiveType == Custom`. Див. нижче. |
| `TimeLimit` | `float` секунд (за замовчуванням `0`) | `0` = без ліміту. Незалежний дедлайн на цей крок, окремий від `TimeLimit` квесту. Див. «Таймерні квести» нижче. |
| `TimerWarningThreshold` | `float` секунд (за замовчуванням `0`) | `0` = без попередження. Скільки секунд має залишитися, щоб спрацював `OnObjectiveTimerWarning` (один раз). Має сенс лише за `TimeLimit > 0`. |
| `CustomData` | `TMap<FName, FString>` | Довільні пари ключ/значення для вашої власної логіки. |
| `ObjectiveIcon` | `UTexture2D*` | Необов'язкова іконка для UI. |

### Типи цілей і відповідна їм подія

| `ObjectiveType` | Збігається з EventID |
|---|---|
| `KillTarget` | `"Kill"` |
| `CollectItem` | `"Collect"` |
| `ReachLocation` | `"Location"` |
| `InteractWith` | `"Interact"` |
| `Custom` | Те, що вказано в `ExpectedEventID` (або будь-яка подія, якщо `ExpectedEventID` не заданий) |

Механіку зіставлення докладно розібрано в [04 — Подієвий прогрес](04-Event-Driven-Progress.md) — там же пояснюється важливий нюанс: дві *різні* цілі (в одному квесті або в різних квестах), що мають однакову пару `(ObjectiveType, ObjectiveIdentifier)`, **обидві** отримають залік від однієї й тієї самої відповідної події. Це усвідомлена поведінка спільного заліку, а не баг — прочитайте той розділ, перш ніж покладатися на глобальну унікальність ідентифікаторів.

### Custom-цілі

`Custom`-цілі існують для геймплею, який не вписується в Kill/Collect/Location/Interact — крафт, набір репутації, перемоги в PvP, будівництво тощо. `ExpectedEventID` звужує, яку категорію подій ціль приймає, тож `Custom`-ціль з ідентифікатором `"HealthPotion"` та `ExpectedEventID = "Craft"` реагує лише на подію крафту, а не на `Collect`-подію з тим самим ідентифікатором:

```cpp
// Ціль: ObjectiveType = Custom, ObjectiveIdentifier = "HealthPotion", ExpectedEventID = "Craft"
UQuestBlueprintLibrary::NotifyCustomQuestEvent(PlayerState, "Craft", "HealthPotion", 1); // зарахується
UQuestBlueprintLibrary::NotifyCollectEvent(PlayerState, "HealthPotion", 1);              // проігнорується
```

Якщо `ExpectedEventID` залишити порожнім, ціль прийме *будь-який* EventID з відповідним ідентифікатором — максимум гнучкості, мінімум захисту від випадкових збігів.

## UQuestData

| Властивість | Тип | Значення |
|----------|------|---------|
| `QuestID` | `FName` | Стабільний ідентифікатор, який використовують `FindQuestByID` і дані збереження. Обов'язково заповніть — див. [09](09-Validation-And-Cheats.md). |
| `QuestName` | `FText` | Відображувана назва. |
| `QuestDescription` | `FText` | Повний наративний текст. |
| `QuestSummary` | `FText` | Стислий опис для списку в журналі квестів. |
| `QuestCategory` | `FName` | Довільний тег-фільтр ("MainStory", "SideQuest", …). |
| `Objectives` | `TArray<UQuestObjectiveData*>` | Цілі, з яких складається квест. |
| `PrerequisiteQuests` | `TArray<UQuestData*>` | Квести, які мають бути завершені раніше. Автоматично перевіряються на цикли — див. [09](09-Validation-And-Cheats.md). |
| `CustomData` | `TMap<FName, FString>` | Довільні метадані (ID NPC, що видає квест, тег сюжетної лінії, …). |
| `Rewards` | `FQuestReward` | Див. нижче. |
| `QuestIcon` | `UTexture2D*` | Іконка для UI. |
| `QuestGiverDisplayName` | `FText` | Лише інформаційно. |
| `bCanAbandon` | `bool` (за замовчуванням `true`) | Чи може гравець вручну скасувати квест. |
| `bAutoComplete` | `bool` (за замовчуванням `false`) | `true`: завершується миттєво, щойно виконані всі обов'язкові цілі. `false`: гравець має активно здати квест (див. [05 — Компоненти сцени](05-World-Components.md), `UQuestReceiverComponent`). |
| `TimeLimit` | `float`, секунди (за замовчуванням `0`) | `0` = без обмеження. Див. «Таймерні квести» нижче. |
| `TimerWarningThreshold` | `float`, секунди (за замовчуванням `0`) | `0` = без попередження. Скільки секунд має залишитися, щоб спрацював `OnQuestTimerWarning` (один раз). Має сенс лише за `TimeLimit > 0`. |
| `QuestSharingMode` | `EQuestSharingMode` | Personal / Shared / Individual — див. [01](01-Core-Concepts.md). |
| `bAutoEnrollNewParticipants` | `bool` (за замовчуванням `true`) | Лише для Shared/Individual — див. [06](06-Multiplayer.md). |
| `MinPartySize` / `MaxPartySize` | `int32` | Обмеження розміру групи (Shared/Individual). `0` означає «без обмеження» для `MaxPartySize`; `MinPartySize`, що дорівнює `0` або `1`, дозволяє соло-старт. |

### FQuestReward

```cpp
USTRUCT(BlueprintType)
struct FQuestReward
{
    TArray<TSubclassOf<AActor>> ItemRewards;
    TMap<FName, int32> RewardAmounts; // наприклад, "Gold" -> 50, "Carrots" -> 12, "ExperiencePoints" -> 100
};
```

Навмисно не має вбудованого поняття валюти, досвіду чи будь-якого іншого іменованого ресурсу — що взагалі таке «нагорода» (золото? морква? репутація? очки навички?) цілком визначається проєктом. Два дженерик-канали покривають будь-яку форму нагороди: `RewardAmounts` — для всього, що виражається іменованим числом, `ItemRewards` — для всього, що є дискретним заспавненим/виданим актором.

`FQuestReward` — це лише *дані*, нічого не витрачається автоматично, і плагін ніяк не читає та не інтерпретує ключі `RewardAmounts` сам. Реальну видачу реалізуйте в `GiveQuestRewards()`, маршрутизуючи кожен ключ туди, де він щось означає у вашій грі; див. [02 — Інтеграція без налаштування](02-Zero-Config-Integration.md).

### Auto-complete проти здачі квесту

- `bAutoComplete = true`: квест завершується (і спрацьовують нагороди) одразу, щойно виконана остання обов'язкова ціль. Добре підходить для простих квестів на збір/убивство.
- `bAutoComplete = false`: квест стає *готовим до здачі* (`IsQuestReadyToTurnIn()` поверне `true`), але лишається `Active`, доки щось не викличе `CompleteQuest()` — як правило, `UQuestReceiverComponent` на NPC чи скрині. Добре підходить для квестів із наративною розв'язкою в конкретному місці.

### Таймерні квести

Установіть `TimeLimit` у додатне число секунд. Після старту `UQuestManagerComponent` кожен тік відлічує `TimeRemaining` для цього квесту. Автоматично відбуваються дві речі:

- `OnQuestTimerWarning(Quest, TimeRemaining)` спрацьовує один раз, коли залишковий час падає до `TimerWarningThreshold` (`0` = без попередження, задається для кожного квесту — єдиного проєктного значення за замовчуванням більше немає) — використовуйте для миготливого попередження в UI.
- Якщо таймер дійде до нуля раніше, ніж квест завершиться, квест **автоматично провалюється** (`FailQuest` викликається за вас).

Квест `DQ_TimedDelivery` на демо-карті (`Content/Demo/DataAssets/Quests/`) — робочий приклад: `TimeLimit = 45`, `bAutoComplete = true`, одна ціль `ReachLocation`.

### Таймерні цілі

Цілі теж можуть нести власні `TimeLimit`/`TimerWarningThreshold`, повністю незалежні від квестових — квест загалом може бути без обмеження за часом і при цьому мати один 30-секундний крок «протримайся на місці», і навпаки. Відлік іде, лише поки конкретно ця ціль `Active`, і починається заново щоразу, коли ціль активується (зокрема й пізніша послідовна).

Що відбувається, коли таймер цілі доходить до нуля, залежить від `bIsOptional`:

- **Обов'язкова** ціль: весь квест провалюється (`FailQuest`), точно так само, як при вичерпанні квестового `TimeLimit`.
- **Необов'язкова** ціль: лише вона позначається як `Failed` — квест триває і все ще може завершитися, щойно виконані обов'язкові цілі (те саме правило `bIsOptional`, що вже виключає її з перевірки завершення).

`OnObjectiveTimerWarning(Quest, Objective, TimeRemaining)` — аналог `OnQuestTimerWarning` на рівні цілі, див. [04 — Подієвий прогрес](04-Event-Driven-Progress.md#підписка-на-прогрес).

Залишковий час і квесту, і будь-якої таймерної цілі видно наживо у вкладці Quests оверлея `QuestSystemDebug` — див. [09 — Валідація та чит-команди](09-Validation-And-Cheats.md).

### Пререквізити та валідація

Будь-який квест може перелічити інші квести в `PrerequisiteQuests`; `CanStartQuest()` відмовиться його запускати, доки не завершений кожен пререквізит. Граф залежностей автоматично перевіряється в редакторі (під час редагування та збереження) на предмет циклів — повний набір інструментів валідації й налагодження див. у [09 — Валідація та чити](09-Validation-And-Cheats.md).

Пререквізити вільно перетинають режими шарингу (наприклад, Personal-квест може вимагати Shared-квест) — завершеність перевіряється на тому менеджері, який реально володіє *конкретно цим пререквізитом*, а не сліпо на менеджері перевірюваного квесту, тож завершення Shared-пререквізиту (відстежуване на party/GameState-менеджері) коректно розпізнається під час перевірки Personal-квесту. Один напрямок плагін не розв'язує сам: **Shared**-квест, що вимагає **Personal/Individual**-пререквізит — питання «чиє особисте завершення має зарахуватися для всієї групи» це рішення геймдизайну, яке плагін лишає вам, тому такий пререквізит перевіряється за власними завершеними квестами party-менеджера (до яких персональний квест ніколи не приєднується) і ніколи не буде виконаний автоматично.

## Квести демо-сцени

`Content/Demo/DataAssets/{Quests,Objectives}/` містить робочий, ігровий набір квестів, що задіює все описане вище — конкретний орієнтир на додачу до таблиць властивостей. `QuestID`/`ObjectiveIdentifier` — стабільні функціональні ключі (за ними працюють `FindQuestByID` і зіставлення ігрових подій відповідно); імена ассетів і відображувані `Name` — косметика, їх можна вільно перейменовувати, не чіпаючи ці поля.

### Квести

| Ассет | `QuestID` | Назва | Категорія | Режим доступу | Цілі |
|---|---|---|---|---|---|
| `DQ_AlchemistApprentice` | `Quest_AlchemistApprentice` | Alchemist's Apprentice | Crafting | Personal | `DO_CraftHealthPotions`, `DO_CraftFungi`, `DO_CraftManaPotion` |
| `DQ_BanditThreat` | `Quest_BanditThreat` | Bandit Threat | MainStory | Personal | `DO_KillBandits` |
| `DQ_CrystalWarmup` | `Quest_CrystalWarmup` | Crystal Warmup | DemoV2 | Personal — пререквізит `DQ_TimedDelivery` | `DO_CollectCrystals` |
| `DQ_HerbGathering` | `Quest_HerbGathering` | Healing Herbs | — | Personal | `DO_CollectHerbs` |
| `DQ_IntroQuest` | `Quest_Intro` | A Troubled Village | MainStory | Personal | `DO_TalkToElder` |
| `DQ_PartyHarvest` | `Quest_PartyHarvest` | Party Harvest | DemoV2 | Shared | `DO_HarvestBerries` |
| `DQ_Pilgrimage` | `Quest_Pilgrimage` | Pilgrimage | DemoV2 | Individual | `DO_ReachShrine` |
| `DQ_TargetPractice` | `Quest_TargetPractice` | Target Practice | DemoV2 | Personal — пререквізит `DQ_CrystalWarmup`, ручна здача | `DO_KillDummies` |
| `DQ_TimedDelivery` | `Quest_TimedDelivery` | Timed Delivery | DemoV2 | Personal — `TimeLimit = 45с` | `DO_ReachDelivery` |
| `DQ_TreasureHunt` | `Quest_TreasureHunt` | The Treasure Hunt | — | Personal — послідовний, ручна здача | `DO_FindMap` → `DO_DecodeMap` → `DO_LocateTreasure` |
| `DQ_UrgentDelivery` | `Quest_UrgentDelivery` | Urgent Delivery | — | Personal — `TimeLimit = 25с` | `DO_DeliverMedicine`, `DO_LocateChild` |

### Цілі

| Ассет | `ObjectiveIdentifier` | Назва | Тип | Ціль |
|---|---|---|---|---|
| `DO_CollectCrystals` | `EnergyCrystal` | Collect Energy Crystals | CollectItem | 3 |
| `DO_CollectHerbs` | `MagicalHerb` | Collect Magical Herbs | CollectItem | 4 |
| `DO_CraftFungi` | `Fungi` | Craft Fungi | Custom (`ExpectedEventID="Craft"`) | 2 |
| `DO_CraftHealthPotions` | `HealthPotion` | Craft Health Potions | Custom (`ExpectedEventID="Craft"`) | 3 |
| `DO_CraftManaPotion` | `ManaPotion` | Craft Mana Potion | Custom (`ExpectedEventID="Craft"`) | 1 |
| `DO_DecodeMap` | `Decoder` | Decode the Map | InteractWith | 1 |
| `DO_DeliverMedicine` | `SickChild` | Deliver Medicine | CollectItem | 2 |
| `DO_FindMap` | `AncientMap` | Find the Ancient Map | CollectItem | 1 |
| `DO_HarvestBerries` | `PartyBerry` | Harvest Party Berries | CollectItem | 6 |
| `DO_KillBandits` | `Bandit` | Defeat Bandits | KillTarget | 5 |
| `DO_KillDummies` | `TrainingDummy` | Destroy Training Dummies | KillTarget | 3 |
| `DO_LocateChild` | `ChildLocation` | Find the Sick Child | ReachLocation | 1 |
| `DO_LocateTreasure` | `TreasureLocation` | Locate the Treasure | ReachLocation | 1 |
| `DO_ReachDelivery` | `DeliveryPoint` | Reach the Delivery Point | ReachLocation | 1 |
| `DO_ReachShrine` | `AncientShrine` | Reach the Ancient Shrine | ReachLocation | 1 |
| `DO_TalkToElder` | `VillageElder` | Speak with the Elder | InteractWith | 1 |

## Куди далі

- Підключення ігрових подій до цих цілей: [04 — Подієвий прогрес](04-Event-Driven-Progress.md)
- Розміщення акторів, що видають квести, на рівні: [05 — Компоненти сцени](05-World-Components.md)

<!-- doc-footer:start -->
---
*Last updated: 2026-08-05 18:13 UTC*
<!-- doc-footer:end -->
