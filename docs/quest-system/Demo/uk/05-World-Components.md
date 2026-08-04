# 05 — Компоненти сцени

*[🇬🇧 English](../en/05-World-Components.md) | 🇺🇦 Українська*

Усе, що дизайнер рівня розміщує у світі — це маленький компонований `UActorComponent` (або крихітний готовий `AActor`, що огортає такий компонент). Прикріплюйте що треба до будь-якого актора — обов'язкового базового класу немає. Усі класи на цій сторінці живуть у модулі `QuestSystemDemoWorld` (окремому від Core-модуля `QuestSystemDemo`, оскільки нічого з цього не належить до базової логіки квестів — список модулів див. у [10 — Інтеграція на C++](10-CPP-Integration.md)); підключіть `DemoQuestSystemWorldTypes.h`, щоб отримати все одразу, або окремі заголовки.

## UDemoQuestGiverComponent

Видає квести гравцям.

| Член | Призначення |
|---|---|
| `AvailableQuests` (`TArray<UDemoQuestData*>`) | Квести, які може запропонувати цей актор. |
| `GiveQuest(Quest, Participants)` | Запускає квест. Маршрутизує автоматично за `QuestSharingMode`: Personal іде `Participants[0]`; Shared/Individual запускають party-квест на весь масив. |
| `CanGiveQuest(Quest, Player)` / `HasAvailableQuestsForPlayer(Player)` | Запити для UI/маркерів. |
| `GetAvailableQuests(Player)` | `BlueprintNativeEvent` — перевизначте, щоб додати свою фільтрацію понад пререквізити. |
| `ShouldOfferQuestToPlayer(Quest, Player)` | `BlueprintNativeEvent`, за замовчуванням `true` — перевизначте для перевірок за рівнем/репутацією/прапорцями. |
| `OnQuestAcceptedByPlayer(Quest, Player)` | `BlueprintNativeEvent` — перевизначте для діалогів/VFX/SFX. |
| `OnQuestAcceptedDelegate` | Multicast-делегат для зовнішніх слухачів (наприклад, маркер квесту — див. `UDemoQuestMarkerComponentBase` нижче). |

```cpp
UDemoQuestGiverComponent* Giver = CreateDefaultSubobject<UDemoQuestGiverComponent>(TEXT("QuestGiver"));
```

## UDemoQuestReceiverComponent

Приймає здачу квестів. Дзеркально до giver'а:

| Член | Призначення |
|---|---|
| `AcceptedQuests` (`TArray<UDemoQuestData*>`) | Квести, які можна здати тут. |
| `TurnInQuestFromPlayer(Player, Quest)` | Завершує квест і запускає нагороди. |
| `CanTurnInQuest(Quest, Player)` / `HasTurnInQuestsForPlayer(Player)` | Запити для UI/маркерів. |
| `GetTurnInQuests(Player)` | `BlueprintNativeEvent` — дзеркало giver'ського `GetAvailableQuests`; перевизначте, щоб додати фільтрацію понад перевірку завершеності. |
| `ShouldAcceptQuestFromPlayer(Quest, Player)` | `BlueprintNativeEvent`, за замовчуванням `true` — перевизначте для своїх умов. |
| `OnQuestTurnedInByPlayer(Quest, Player)` | `BlueprintNativeEvent` — перевизначте для святкових ефектів. |
| `OnQuestTurnedInDelegate` | Multicast-делегат для зовнішніх слухачів. |

## Квестові маркери

Керують візуальним індикатором («!» / «?») над giver'ом/receiver'ом/target'ом. Це невелика ієрархія класів, а не один компонент із перемикачем режиму:

- **`UDemoQuestMarkerComponentBase`** (абстрактний) — усе спільне. Сам знаходить сусідній `UDemoQuestGiverComponent` / `UDemoQuestReceiverComponent` / `UDemoQuestTargetComponent` на тому самому акторі, відстежує стан квестів для локального гравця й віддає `GetMarkerState(Player)` / `GetRelevantQuest(Player)` (конкретний квест, що визначає стан — той, що готовий до здачі, або доступний — корисно для тултипів/віджетів, яким треба назвати квест, а не просто показати стан) / `GetLinkedGiverComponent()` / `GetLinkedReceiverComponent()` / `GetLinkedTargetComponent()`, плюс Blueprint-подію `OnMarkerStateChanged(NewState, OldState, RelevantQuest)`, яка спрацьовує при будь-якій зміні стану, у будь-якому нащадку. Увімкніть `bUseEventDrivenUpdates` (за замовчуванням `true`, рекомендовано), щоб оновлення йшло лише при реальних змінах, а не опитуванням за таймером — підписка йде **на обидва** менеджери, особистий і party/GameState, тож Shared- та Individual-квести оновлюють маркер так само швидко, як Personal (див. [маршрутизацію до потрібного менеджера](06-Multiplayer.md)). Для world-цілі (див. вище) маркер з'являється лише поки у гравця активна ціль, яку ця ціль просуває, і знову ховається, коли ціль виконана. `bTrackInProgressQuests` (за замовчуванням вимкнено) показує стан `InProgress` для квесту, пов'язаного саме з цим актором — виданого його Giver'ом або прийманого його Receiver'ом, залежно від того, що є, — який у гравця зараз активний, але не завершений; працює однаково і для суто giver-актора на кшталт `ADemoQuestBoard`, і для receiver-актора, і для комбо. Також за замовчуванням ігнорує масштаб батька (`bAbsoluteScale`, успадковано від `USceneComponent`, увімкнено за замовчуванням у конструкторі базового класу) — оскільки маркер зазвичай кріпиться до меша самого пропа, більший/менший екземпляр пропа не роздуває і не стискає разом із собою візуал маркера. Зніміть галку **Absolute Scale** (категорія Transform у панелі Details) на самому компоненті, якщо колись знадобиться зворотне — успадкування масштабу.
- **`UDemoQuestStaticMeshMarkerComponent`** — вбудований нащадок на мешах: призначте `MarkerMeshComponent` (нормальний picker компонента — вибір наявного `StaticMeshComponent` на цьому ж акторі за іменем) плюс `AvailableMaterial` / `TurnInMaterial` / (опційно) `InProgressMaterial`, і він сам змінює матеріали та видимість. Якщо `MarkerMeshComponent` не заданий, спрацьовує фолбек: спочатку дочірній компонент з іменем рівно `"MarkerMesh"`, потім перший `StaticMeshComponent` серед дочірніх саме цього компонента (працює і для компонента, доданого в Blueprint, незалежно від імені), і нарешті — якщо увімкнено `bAutoCreateMarkerVisual` — автостворення в `BeginPlay` (використовуючи `DefaultMarkerMesh`, за замовчуванням інженерний конус); вимкніть `bAutoCreateMarkerVisual`, якщо замість тихого дефолту хочете попередження в лозі. Хоч би який шлях спрацював, підсумковий використовуваний меш доступний лише для читання як `MarkerMesh`.
- **`UDemoQuestWidgetMarkerComponent`** — вбудований нащадок на віджетах: призначте `WidgetClass` (`TSubclassOf<UUserWidget>` — будь-який Widget Blueprint або нативний `UUserWidget`), і компонент сам покаже/сховає його, але **ніяк не впливає на його зовнішній вигляд** — на відміну від слотів матеріалів у мешевого маркера, тут немає вбудованої властивості тексту/іконки/кольору: усе це цілком усередині `WidgetClass`. Реалізуйте на класі віджета `IDemoQuestMarkerVisualInterface` (нижче), щоб дізнаватися, *чому* він показаний (новий стан і конкретний квест, який його визначає), і реагувати як завгодно — перемкнути bound-зображення, змінити текст, програти анімацію. Його `WidgetComponent` розв'язується тим самим трирівневим способом, що й `MarkerMeshComponent` у мешевого маркера: спочатку явне посилання через picker `MarkerWidgetComponent`, потім дочірній компонент з іменем `"MarkerWidget"`, потім перший `WidgetComponent` серед дочірніх цього компонента, і нарешті (якщо увімкнено `bAutoCreateMarkerVisual`) автостворення — налаштовується через `WidgetSpace` / `DrawSize` / `Pivot` для автоствореного випадку (за замовчуванням Screen space; перемкніть на World для білборд-варіанта в 3D-просторі).
- **`UDemoQuestBlueprintMarkerComponent`** — спавнить і прикріплює цілий актор (`MarkerActorClass`, зазвичай Blueprint — плавуча стрілка, анімована іконка, партикли, будь-що, що можна зібрати в Actor Blueprint) замість меша чи віджет-компонента. Сам перемикає видимість заспавненого актора, а якщо цей актор реалізує `IDemoQuestMarkerVisualInterface` — прокидає йому зміну стану теж, тож він може не лише показати/сховати себе, а й програти анімацію, змінити матеріал тощо. Це той варіант, який берете, коли маркер має бути «цілим актором», а не мешем чи віджетом — без виходу за межі плагіна. А оскільки `MarkerActorClass` — це `TSubclassOf<AActor>`, такий Blueprint може додати собі `Widget Component` прямо у своїй же панелі компонентів (білборд-іконка на UMG) **взагалі без жодного рядка C++**: Blueprint-ассет не компілюється проти модуля плагіна, тому може використовувати будь-який увімкнений у проєкті плагін (зокрема й UMG) незалежно від того, які C++ модулі взагалі існують. Повний робочий приклад цього патерну — пара `AQuestDemoMarkerWidgetActor` + `UQuestDemoMarkerWidget` у демо-хості: нативний C++-реалізатор інтерфейсу, що керує суто C++ UMG-віджетом (без WBP-ассету), який показує стан маркера й назву квесту. Призначте `AQuestDemoMarkerWidgetActor` у `MarkerActorClass`, щоб побачити його наживо, або скопіюйте як зразок для суто Blueprint-варіанта. Але якщо маркер має бути *лише* віджетом — беріть `UDemoQuestWidgetMarkerComponent` вище: він розв'язує ту саму задачу, не спавнячи окремий актор.

**Керування маркером вручну.** База також віддає назовні механіку оновлення, доступну будь-якому нащадку: `UpdateMarker(Player)` перераховує стан просто зараз, `SetMarkerState(State)` виставляє стан напряму (в обхід автовизначення — для катсцен і скриптових сцен), а `StartAutoUpdate()` / `StopAutoUpdate()` керують таймером опитування в рантаймі. Сам таймер налаштовується через `UpdateInterval` (секунди, `0` = не опитувати) і `bAutoUpdate`; за увімкненого `bUseEventDrivenUpdates` опитування працює лише підстрахуванням, тому збільшити `UpdateInterval` — або взагалі вимкнути `bAutoUpdate` і викликати `UpdateMarker()` самостійно — найдешевший варіант для рівня з великою кількістю маркерів.

`IDemoQuestMarkerVisualInterface` (`Interfaces/DemoQuestMarkerVisualInterface.h`) — спільний контракт за останніми двома пунктами: один `BlueprintNativeEvent`, `OnQuestMarkerStateChanged(NewState, RelevantQuest)`, який може реалізувати будь-що, що візуалізує стан — актор (для `UDemoQuestBlueprintMarkerComponent`) або `UUserWidget` (для `UDemoQuestWidgetMarkerComponent`).

Перемикача режимів немає, тож ніщо не заважає додати **кілька маркерних компонентів на один актор** — наприклад, конус `UDemoQuestStaticMeshMarkerComponent` і віджет-табличку `UDemoQuestWidgetMarkerComponent` одночасно, кожен незалежно відстежує той самий стан квесту й малює своє.

### Свій тип маркера

Успадковуйтеся від `UDemoQuestMarkerComponentBase` і перевизначте два protected-віртуальні методи: `SetupVisualization(Owner)` (викликається один раз із `BeginPlay`, після того як сусіди Giver/Receiver/Target уже закешовані — знайдіть або створіть потрібний візуальний компонент) та `ApplyVisualization(State, RelevantQuest)` (викликається при кожній зміні стану — оновіть візуал). Вихідний код усіх трьох вбудованих нащадків — зразки цього патерну; якщо потрібно просто «без візуалу, лише `OnMarkerStateChanged`» — одноразовому Blueprint-нащадку `UDemoQuestMarkerComponentBase` не потрібен жоден із них.

Нативному C++-нащадку з типізованою властивістю `UWidgetComponent`/`UUserWidget` потрібна залежність від `UMG` — саме це вже являє собою `UDemoQuestWidgetMarkerComponent` вище, що живе в модулі `QuestSystemDemoWorld` (який залежить від `UMG`, на відміну від core-модуля `QuestSystemDemo`). Пишіть власний нащадок лише тоді, коли цей вбудований компонент справді не підходить — наприклад, маркер, що керує чимось більшим, ніж один меш чи віджет-компонент.

## UDemoQuestInteractorComponent — сторона гравця

Прикріпіть до пішака гравця, прив'яжіть один input action до `TryInteract()` — і все готово:

```cpp
QuestInteractor = CreateDefaultSubobject<UDemoQuestInteractorComponent>(TEXT("QuestInteractor"));

// у SetupPlayerInputComponent, будь-яка система вводу:
PlayerInputComponent->BindKey(EKeys::E, IE_Pressed, this, &AMyCharacter::Interact);

void AMyCharacter::Interact() { QuestInteractor->TryInteract(); }
```

`TryInteract()` безпечно викликати на клієнті (перекидається на сервер через RPC). На сервері він сканує `InteractionRadius` сантиметрів навколо пішака і, за пріоритетом, діє на найближчому акторі, який щось пропонує:

1. `UDemoQuestReceiverComponent` з квестом, готовим до здачі → здає його.
2. `UDemoQuestGiverComponent` з доступним квестом → приймає його (Personal-квести йдуть гравцеві, який взаємодіє; Shared/Individual-квести запускаються на всіх поточних підключених гравців — див. [06](06-Multiplayer.md)).
3. `UDemoQuestTargetComponent`, що відповідає на тригер Interact → генерує подію цього компонента.

`OnQuestInteractionPerformed(TargetActor, Quest)` спрацьовує на сервері після успішної взаємодії (`Quest` дорівнює null для простої interaction-цілі).

## UDemoQuestTargetComponent — єдина розумна world-ціль

Повісьте це на **будь-який** актор (пляшку, підбираний предмет, ворога, NPC, проп) і налаштуйте в панелі Details. Це єдиний механізм для «цей актор — ціль квесту», побудований навколо трьох питань:

**1. Що він повідомляє?** — квестову подію, тобто стандартний провід `NotifyQuestEvent`:

| Член | Призначення |
|---|---|
| `SourceObjective` | **Опційно.** Зв'яжіть з `UDemoQuestObjectiveData`, яку обслуговує ця ціль — три поля нижче заповняться з неї автоматично й **заблокуються** (стануть сірими): не треба вбивати рядок руками, і ідентичність не розійдеться з ассетом цілі. Зручність лише в редакторі; рантайм використовує скопійовані значення. Лишіть порожнім, щоб задати поля вручну. |
| `EventType` | Випадний список: Collect / Kill / Interact / Location / Custom (мапиться на канонічні `DemoQuestEvents::` — обираєте дієслово, а не магічний рядок). |
| `CustomEventID` | Лише за `EventType = Custom` (збігається з `ExpectedEventID` цілі типу Custom). |
| `TargetID` | Зіставляється з `ObjectiveIdentifier` цілі — єдиний рядок, який має збігатися. |
| `Count` | Прогрес за одне спрацювання (за замовчуванням `1`). |

**2. Як він спрацьовує?** — `TriggerSources` це бітова маска; комбінуйте як треба, а ручний `Trigger(PlayerState)` працює завжди поверх:

| Прапорець | Спрацьовує, коли… |
|---|---|
| `Overlap` | гравець наступає на ціль (для вас спавниться сфера-overlap для пішаків радіусом `OverlapRadius`). |
| `Damage` | власник отримує стандартний `ApplyDamage` (вистрілити / вдарити). Ручного захисту від подвійного рахунку не потрібно — див. `ConsumeMode`. |
| `Interact` | гравець використовує `UDemoQuestInteractorComponent` (клавіша E) поруч. |
| `Destroyed` | власник знищений (зараховується його instigator'у, якщо він є). |
| *(ручний)* | ваш власний код бою/інвентарю викликає `Trigger(PlayerState)`. |

**3. Що відбувається після?** — реакція:

| Член | Призначення |
|---|---|
| `ConsumeMode` | `Once` (спрацювати один раз — вбудований захист від подвійного рахунку), `Respawn` (+`RespawnTime`: сховатися, потім з'явитися), або `Repeatable` (фармиться). |
| `bHideOwnerOnConsumed` / `bDestroyOwnerOnConsumed` | Опційне прибирання після спрацювання `Once`. |
| `EffectToSpawn` | Опційний актор, що спавниться біля цілі для фідбеку (уламки, VFX, SFX). |
| `OnQuestTargetTriggered(By)` | **BlueprintImplementableEvent на самому акторі-цілі** — своя реакція (сховати предмет у руках, утекти, програти репліку) без будь-якої проводки на боці гравця. |

Server-authoritative: кожен шлях обробляється лише там, де у власника є authority (власник має реплікуватися, щоб hide/destroy дійшли до клієнтів). Повідомлювана подія так само йде звичайним маршрутом `NotifyQuestEvent`, тому механіка спільного заліку не змінюється.

```
// Пляшка, по якій стріляєш:  EventType=Kill,  TargetID="Bottle",  Trigger=Damage,   Consume=Once + Destroy
// Лист на підлозі (наступив): EventType=Collect, TargetID="Letter", Trigger=Overlap, Consume=Once + Destroy
// NPC, якого грабуєш (E):     EventType=Collect, TargetID="Letter", Trigger=Interact, Consume=Once
//   → реакція в OnQuestTargetTriggered на NPC
```

Оскільки тригер не залежить від повідомлюваної події, листи, що лежать на підлозі (Overlap), і листи у NPC (Interact) можуть слати одну й ту саму подію `Collect`/`"Letter"` і живити одну ціль.

## ADemoQuestLocationTrigger

Готовий актор для цілей `ReachLocation`. Розмістіть на рівні, задайте `LocationID`, і box-overlap повідомить `NotifyLocationReachedEvent` для того гравця, який до нього увійде. `bDestroyAfterActivation` перетворює його на одноразовий тригер замість повторно використовуваного чекпоінта.

## ADemoQuestTargetActor

Готовий розміщуваний актор «з коробки»: видимий меш, `UDemoQuestTargetComponent` і `UDemoQuestStaticMeshMarkerComponent` (сам створює собі іконку, див. вище) — готовий до розміщення на рівні та налаштування в панелі Details. Беріть його, коли ціль — це окремий проп, який ви ставите суто заради квесту (підбираний предмет, розбиваний об'єкт, точка дотику). Коли ціль — це те, що вже є у грі (ворог, NPC), додавайте `UDemoQuestTargetComponent` (і, якщо потрібен маркер, будь-який із нащадків маркера вище) прямо на того актора — цей актор лише зручна обгортка.

Якщо у вашій грі вже є власні системи бою чи інвентарю, можна взагалі обійтися без world-акторів і викликати `NotifyKillEvent` / `NotifyCollectEvent` (або `UDemoQuestTargetComponent::Trigger`) напряму з цього коду.

## ADemoQuestBoard / ADemoQuestChest

Ще два готові розміщувані актори «з коробки» — для пропа лише-giver або лише-receiver: видимий меш плюс `UDemoQuestGiverComponent` (`ADemoQuestBoard`, дошка оголошень) або `UDemoQuestReceiverComponent` (`ADemoQuestChest`, точка здачі), плюс `UDemoQuestStaticMeshMarkerComponent` — готові до розміщення на рівні та налаштування в панелі Details. Як і весь інший `Actors/`, без контенту: `PropMesh` без меша за замовчуванням, призначте свій на конкретному екземплярі або в Blueprint-нащадку; маркер сам створить собі іконку, якщо ви і його не призначите (див. [Квестові маркери](#квестові-маркери) вище). `AvailableQuests` / `AcceptedQuests` налаштовуються прямо на giver/receiver-компоненті.

## Складання свого NPC

Ніщо з переліченого вище не потребує спеціального базового класу — комбінуйте потрібні компоненти на будь-якому `AActor` чи `ACharacter`. NPC, який і видає, і приймає квести — це просто `Character` з доданими `UDemoQuestGiverComponent` + `UDemoQuestReceiverComponent` + `UDemoQuestStaticMeshMarkerComponent` — готового класу під це в плагіні немає (на відміну від `ADemoQuestBoard`/`ADemoQuestChest` вище), бо NPC потрібен скелетний меш і анімації — це неминуче контент конкретної гри, а не плагіна. Зберіть його як Blueprint (або як тонкий native-нащадок `ACharacter`, якщо надаєте перевагу C++) у своєму проєкті.

Відкрийте `DemoMap` у редакторі, щоб побачити `ADemoQuestBoard`/`ADemoQuestChest` і демо-NPC на Blueprint'і, налаштовані на реальні квести.

## Куди далі

- Як giver/receiver маршрутизують Personal проти Shared/Individual квестів під капотом: [06 — Мультиплеєр](06-Multiplayer.md)


---
*Generated 2026-08-04 14:54 UTC from `Docs/Full/` - do not edit this page directly.*
