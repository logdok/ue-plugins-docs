# Довідник API

*[🇬🇧 English](../en/API-Reference.md) | 🇺🇦 Українська*

Сторінка перелічує публічні класи InteractionSystem 1.0: що вони роблять, які мають властивості й початкові значення, де викликати функції та на якій машині спрацьовують події. Відстані — у сантиметрах, кути — у градусах, швидкості — у см/с і градусах/с.

Позначки в таблицях:

- **BP** — доступно в Blueprint (у дужках: `RW` — читання й запис, `RO` — лише читання);
- **C++** — лише в C++;
- **Сервер** — функція діє лише на authority, на клієнті нічого не робить;
- **Локально** — подія або функція виконується лише на машині гравця, що керує pawn.

| Заголовок | Модуль | Вміст |
|---|---|---|
| `InteractorComponent.h` | `InteractionSystem` | `UInteractorComponent`, делегати фокуса й результату |
| `InteractableInterface.h` | `InteractionSystem` | `IInteractableInterface` |
| `InteractionSystemTypes.h` | `InteractionSystem` | `EInteractionAction`, `EInteractionRejectReason`, `FInteractionFilter`, `FInteractionRules` |
| `InteractionSystemSettings.h` | `InteractionSystem` | `UInteractionSystemSettings` — мережеві межі проєкту |
| `InteractionGripTypes.h` | `InteractionSystem` | `EInteractionHand`, `EInteractionReleaseReason`, `FInteractionGripMath` |
| `InteractionGrabTargetComponent.h` | `InteractionSystem` | `UInteractionGrabTargetComponent` |
| `InteractionBlueprintGrabTargetComponent.h` | `InteractionSystem` | `UInteractionBlueprintGrabTargetComponent` |
| `InteractionClaimSubsystem.h` | `InteractionSystem` | `UInteractionClaimSubsystem` |
| `InteractionGameplayTags.h` | `InteractionSystem` | `InteractionGameplayTags::Item`, `Locked` |
| `InteractionManipulationComponent.h` | `InteractionSystemWorld` | `UInteractionManipulationComponent` |
| `InteractionManipulationProfile.h` | `InteractionSystemWorld` | `FInteractionManipulationSettings`, `UInteractionManipulationProfile`, переліки руху |
| `InteractionGripPointComponent.h` | `InteractionSystemWorld` | `UInteractionGripPointComponent` |
| `InteractionDockComponent.h` | `InteractionSystemWorld` | `UInteractionDockComponent` |
| `InteractionBowComponent.h` | `InteractionSystemWorld` | `UInteractionBowComponent` |
| `InteractionStrikeComponent.h` | `InteractionSystemWorld` | `UInteractionStrikeComponent` |
| `InteractionDestructibleComponent.h` | `InteractionSystemWorld` | `UInteractionDestructibleComponent` |
| `InteractionMeshComponent.h` | `InteractionSystemWorld` | `UInteractionMeshComponent` |
| `InteractionPhysicsActor.h` | `InteractionSystemWorld` | `AInteractionPhysicsActor` |
| `InteractionMasterItem.h` | `InteractionSystemWorld` | `AInteractionMasterItem` |
| `InteractionSpearActor.h` | `InteractionSystemWorld` | `AInteractionSpearActor` |
| `InteractionPhysicsDataAsset.h` | `InteractionSystemWorld` | `UInteractionPhysicsDataAsset` |
| `InteractionDebugLibrary.h` | `InteractionSystemDebug` | `UInteractionDebugLibrary` |
| `InteractionCheatManagerExtension.h` | `InteractionSystemDebug` | Консольні команди налагодження |

## Ядро: InteractionSystem

### UInteractorComponent

Scene component на pawn: шукає ціль, показує локальний фокус і надсилає дії на сервер. Один компонент — одне джерело дії: камера на desktop або одна VR-рука. Компонент реплікується; RPC ідуть через pawn, яким володіє клієнт.

| Властивість | Початково | Доступ | Призначення |
|---|---|---|---|
| `bUsePawnViewpoint` | `true` | BP RW | Щокадру переносити компонент у точку огляду контролера. Для VR-руки вимкніть |
| `TraceDistance` | 300 | BP RW | Довжина променя |
| `TraceRadius` | 0 | BP RW | 0 — line trace; більше 0 — sphere sweep з перевіркою видимості контакту |
| `FocusUpdateHz` | 30 | BP RW | Частота пошуку фокуса цього джерела |
| `PoseReplicationHz` | 30 | BP RW | Частота спільного пакета поз (1–120); діє значення першого активного interactor pawn |
| `bNearInteraction` | `false` | BP RW | Шукати ціль overlap-сферою навколо долоні (VR) замість променя |
| `NearRadius` | 15 | BP RW | Радіус ближнього пошуку |
| `Hand` | `Any` | BP RW | Сторона руки для авторських точок хвату |
| `PalmOffset` | Identity | BP RW | Калібрування долоні відносно компонента |
| `TrackingLossGrace` | 0.25 с | BP RW | Скільки терпіти втрату трекінгу, перш ніж відпустити предмет без кидка |
| `TraceChannel` | `Visibility` | BP RW | Канал пошуку й перевірки перешкод; цілі мають його блокувати |
| `Filter` | порожній | BP RW | Вимоги до тегів цілі, див. [FInteractionFilter](#finteractionfilter) |
| `GrabOffset` | (0, 0, 0) | BP RO | Локальний зсув утримуваного предмета від джерела; для VR — нуль |
| `MaxOriginDistance` | 500 | BP RO | Найбільша відстань повідомленої пози від pawn |
| `MinActionInterval` | 0.08 с | BP RO | Мінімальний інтервал між діями, які приймає сервер |
| `PoseTimeout` | 0.75 с | BP RO | Без жодного зразка пози довше за цей час сервер відпускає предмет віддаленого джерела |
| `MaxThrowSpeed` | 2500 | BP RO | Сервер обрізає лінійну швидкість кидка |
| `MaxAngularSpeed` | 3600 | BP RO | Сервер обрізає кутову швидкість кидка |

П'ять мережевих меж діють лише тоді, коли ввімкнено відповідний прапорець `bOverrideMaxOriginDistance`, `bOverrideMinActionInterval`, `bOverridePoseTimeout`, `bOverrideMaxThrowSpeed` чи `bOverrideMaxAngularSpeed`; інакше компонент бере значення з [UInteractionSystemSettings](#uinteractionsystemsettings).

| Функція | Доступ | Де викликати | Дія |
|---|---|---|---|
| `RequestGrab()` | BP | Локально | Запит захоплення поточної цілі |
| `RequestRelease(Linear, Angular)` | BP | Локально | Відпустити предмет; нульові вектори — без кидка |
| `RequestInteraction()` | BP | Локально | Звичайна дія з ціллю; якщо рука тримає предмет — його `Use` |
| `CancelGrip(Reason)` | BP | Локально або сервер | Безпечно завершити хват без кидка; `Released` замінюється на `Cancelled` |
| `SetTrackingValid(bValid)` | BP | Локально, щокадру | Повідомити, чи надійна поза пристрою |
| `RefreshFocus()` / `ClearFocus()` | BP | Локально | Оновити або прибрати фокус негайно |
| `GetFocusedActor()` | BP | Локально | Поточна ціль фокуса або `None` |
| `GetHeldActor()` | BP | Будь-де | Підтверджений сервером утримуваний актор |
| `CanFocusNewTarget()` | BP | Локально | Рука активна, відстежується й вільна |
| `IsLocallyControlled()` | BP | Будь-де | Pawn керується на цій машині |
| `GetGripTransform()` | BP | Будь-де | Калібрована поза долоні з одиничним масштабом |
| `HasRequestedGrip()` | BP | Локально | Є утримуваний предмет або запит, що очікує відповіді |
| `HasValidTracking()` | BP | Будь-де | Поточний стан трекінгу джерела |
| `GetMaxOriginDistance()`, `GetMinActionInterval()`, `GetPoseTimeout()`, `GetMaxThrowSpeed()`, `GetMaxAngularSpeed()` | BP | Будь-де | Фактична межа: перевизначення компонента або значення проєкту |
| `TraceTarget(Origin, Rotation, Action)` | C++ | Будь-де | Пошук цілі за тими самими правилами, що й фокус і сервер |
| `CancelTarget(Target, Reason)` | C++ | Сервер | Відпустити саме цю ціль, якщо рука її тримає |

| Подія | Де спрацьовує | Параметри |
|---|---|---|
| `OnFocusChanged` | Локально | `PreviousActor`, `NewActor` |
| `OnActionResult` | На клієнті, що надіслав запит, і на локальному authority | `TargetActor`, `Action`, `bAccepted`, `Reason` — [причина відмови](#переліки-ядра) |

### IInteractableInterface

Інтерфейс цілі для звичайної дії та локального відгуку. Реалізуйте його в Blueprint (**Class Settings → Interfaces**) або C++.

| Функція | Де виконується | Початкова реалізація | Відповідальність |
|---|---|---|---|
| `CanInteract(Interactor, Action)` | Клієнт і сервер | `true` | Дозволити дію; без побічних ефектів |
| `GetInteractionTags()` | Клієнт і сервер | порожній контейнер | Gameplay Tags для фільтра |
| `Interaction(Interactor)` | Сервер | — | Ігрова дія: двері, кнопка, перемикач |
| `OnFocusStart(Interactor)` / `OnFocusEnd(Interactor)` | Локально | — | Візуальний відгук для конкретної руки |
| `OnGrab(Interactor)` | Сервер | — | Ядро викликає, коли ціль бере перша рука |
| `OnRelease(Interactor, Reason, LinearVelocity, AngularVelocity)` | Сервер | — | Ядро викликає, коли відпустила остання рука; швидкості ненульові лише для `Released` |

`Interaction`, `OnFocusStart`, `OnFocusEnd`, `OnGrab` і `OnRelease` — події, які реалізують, але не викликають з Blueprint: так клієнт не обійде сервер і реєстр захоплень. `CanInteract` і `GetInteractionTags` можна викликати.

### FInteractionFilter

| Поле | Значення |
|---|---|
| `RequiredTags` | Ціль має містити всі ці Gameplay Tags (з ієрархією) |
| `BlockedTags` | Будь-який із цих тегів забороняє ціль |
| `RequiredActorTags` | Усі ці імена мають бути в `Actor.Tags` (точний збіг) |

`Matches(Tags, ActorTags)` повертає `true`, якщо всі вимоги виконано. Порожні списки нічого не обмежують.

### FInteractionRules

Статичні перевірки без звернень до світу, спільні для компонента й тестів:

| Функція | Повертає `true`, якщо |
|---|---|
| `IsPoseInReach(Origin, Rotation, PawnLocation, MaxOffset)` | Значення скінченні й `Origin` не далі за `MaxOffset` від pawn |
| `IsFiniteVelocity(Linear, Angular)` | Обидві швидкості скінченні |

### UInteractionGrabTargetComponent

Абстрактна межа між interactor і способом руху предмета. Готова реалізація — [UInteractionManipulationComponent](#uinteractionmanipulationcomponent); реалізація з правилами в Blueprint — [UInteractionBlueprintGrabTargetComponent](#uinteractionblueprintgrabtargetcomponent). Власну реалізацію на C++ показано в [12](12-Cpp-Integration.md#власна-ціль-захоплення).

| Член | Доступ | Призначення |
|---|---|---|
| `TargetTags` | BP RW | Теги для фільтра, якщо актор не реалізує інтерфейс |
| `CanBeginGrip(Hand) const` | C++, virtual | Чи можна почати хват; викликається і локально для фокуса, і на сервері. Початково `false` |
| `GetMaximumHands() const` | C++, virtual | Скільки рук одного pawn допускає ціль. Початково 1 |
| `BeginGrip(Hand)` | C++, virtual, сервер | Після резервування; `false` скасовує резерв. Початково `false` |
| `EndGrip(Hand, Reason, Linear, Angular)` | C++, virtual, сервер | Прибрати лише цю руку; вимушене завершення не повинно кидати предмет |
| `Use(Hand)` | C++, virtual, сервер | Дія руки, що тримає предмет. Початково `false` |

### UInteractionBlueprintGrabTargetComponent

Ціль захоплення, правила якої задає Blueprint-нащадок цього компонента. Компонент сам перевіряє `bGrabEnabled`, вільне місце й те, що друга рука належить тому самому pawn, а потім питає ваші події. Рух власника виконує ваш Blueprint. Компонент реплікується.

| Член | Доступ | Призначення |
|---|---|---|
| `MaximumHands` | BP RO, 1–2 | Скільки рук одного pawn можуть тримати власника |
| `bGrabEnabled` | BP RO, реплікується | Чи приймаються нові хвати; змінюйте через `SetGrabEnabled` |
| `SetGrabEnabled(bEnabled)` | BP, сервер | Увімкнути чи вимкнути захоплення; вимкнення відпускає поточні руки з `Cancelled` |
| `CancelAllGrips()` | BP, сервер | Відпустити всі руки з `Cancelled` |
| `GetHands()` / `GetHandCount()` | BP, реплікується | Руки, що тримають власника, у порядку захоплення |
| Подія `Can Begin Grip(Hand)` | Клієнт і сервер | Додаткове правило допуску; лише реплікований стан. Початково `true` |
| Подія `Begin Grip(Hand)` | Сервер | Рука допущена й зарезервована; `false` скасовує хват (`Unavailable`). Початково `true` |
| Подія `End Grip(Hand, Reason, Linear, Angular)` | Сервер | Рука відпустила власника |
| Подія `Use(Hand)` | Сервер | Рука, що тримає власника, натиснула «використати»; `true` — оброблено. Початково `false` |

### UInteractionSystemSettings

**Project Settings → Plugins → Interaction System** (`Config/DefaultGame.ini`, секція `[/Script/InteractionSystem.InteractionSystemSettings]`). Типові мережеві межі для всіх `UInteractorComponent`, якщо компонент не перевизначає значення.

| Поле | Початково | Призначення |
|---|---|---|
| `MaxOriginDistance` | 500 см | Найбільша відстань повідомленої пози від pawn |
| `MinActionInterval` | 0.08 с | Мінімальний час між прийнятими запитами одного джерела |
| `PoseTimeout` | 0.75 с | Скільки чекати на будь-який зразок пози віддаленого джерела |
| `MaxThrowSpeed` | 2500 см/с | Найбільша лінійна швидкість кидка |
| `MaxAngularSpeed` | 3600 градусів/с | Найбільша кутова швидкість кидка |

### UInteractionClaimSubsystem

World subsystem — серверний реєстр права на ціль. Запити доступні в Blueprint, зміни — лише в C++; на клієнті реєстр порожній.

| Функція | Призначення |
|---|---|
| `TryClaim(Target, Interactor, MaximumHands)` | Додати руку до хвату того самого pawn у межах ліміту (1 або 2) |
| `ReleaseClaim(Target, Interactor)` | Зняти лише цю руку |
| `FindClaimant(Target)` / `FindHands(Target)` | BP. Перша рука або всі руки |
| `IsClaimed(Target)` | BP. Є рука або пов'язаний резерв |
| `FindOwningPawn(Target)` | BP. Pawn-власник хвату чи резерву |
| `GetClaimedTargets()` | BP. Усі цілі, які тримають або резервують |
| `FindLinkedParent(Target)` | BP. Батьківський предмет пов'язаного резерву, наприклад лук для стріли |
| `ReserveLinked(Target, Parent, Pawn)` | Пов'язати залежну ціль (стрілу) з утримуваним батьком того самого pawn; цикли заборонено |
| `ReleaseLinked(Target, Parent)` | Зняти пов'язаний резерв |
| `ForgetTarget(Target)` | Забути ціль і рекурсивно її залежні резерви |

Зазвичай достатньо читати власника; змінюйте реєстр лише через `RequestGrab`/`RequestRelease`.

### FInteractionGripMath

Детермінована математика хвату без звернень до світу: `MakePairFrame` будує систему координат пари долонь, `Capture` фіксує baseline, `Solve` обчислює позу для однієї чи двох рук із масштабом і pivot, `PlaneAngle` повертає кут точки навколо осі. Використовується маніпуляцією й тестами.

### Переліки ядра

| Перелік | Значення |
|---|---|
| `EInteractionAction` | `Interact` — використати; `Grab` — взяти; `Release` — відпустити |
| `EInteractionHand` | `Any` — без сторони (desktop); `Left`; `Right` |
| `EInteractionReleaseReason` | `Released` — гравець відпустив (лише ця причина дозволяє кидок); `TrackingLost`; `Cancelled`; `TargetInvalid`; `Recentered` |
| `EInteractionRejectReason` | `None` — прийнято; `Cooldown`; `InvalidTarget`; `InvalidPose`; `NotReachable`; `HandBusy`; `Claimed`; `Unavailable`. Значення й порядок перевірок — у [13](13-Network-Model.md#причини-відмови) |

### InteractionGameplayTags

| Змінна C++ | Тег | Призначення |
|---|---|---|
| `InteractionGameplayTags::Item` | `Interaction.Item` | Позначає предмет для `RequiredTags` |
| `InteractionGameplayTags::Locked` | `Interaction.Locked` | Типовий тег для `BlockedTags` |

## World: InteractionSystemWorld

### UInteractionManipulationComponent

Готова ціль захоплення для будь-якого актора: одна чи дві руки, авторські точки, вільний хват, масштаб, фізичне утримання, обертові й лінійні ручки, зокрема для іменованої частини актора. Компонент реплікує один атомарний snapshot (руки, опора хвату, квантована поза з масштабом, відставання фізичного тіла, прикріплення, зовнішній драйвер, хід ручки), підтримує push model і на сервері вимикає `ReplicateMovement` власника. У спокої компонент тікає рідко й прокидається від руху.

| Властивість | Початково | Доступ | Призначення |
|---|---|---|---|
| `Profile` | `None` | BP RW | Спільний профіль; якщо порожній — `InlineSettings` |
| `InlineSettings` | див. нижче | BP RW | Налаштування цього екземпляра |
| `TargetComponentName` | `None` | BP RW | Ім'я компонента, яким рухати; `None` — корінь актора. Іменована частина залишається під своїм батьком |
| `bStartWithPhysics` | `false` | BP RW | Симуляція на сервері з початку гри |
| `bGrabEnabled` | `true` | BP RW, реплікується | Вимкнення відхиляє нові хвати й завершує поточні; змінюйте на сервері або через `SetGrabEnabled` |
| `RemoteInterpolationSpeed` | 20 | BP RW | Згладжування snapshot на клієнтах; 0 — без згладжування |
| `bPredictPhysicsForOwner` | `true` | BP RW | Передбачати предмет `Physics` у руці власника |
| `OwnerPredictionTolerance` | 10 | BP RW | Відставання серверного тіла, після якого власник бачить серверну позу |
| `IdleUpdateInterval` | 0.5 с | BP RW | Інтервал тіку в спокої; рух і пробудження тіла повертають щокадровий тік |

| Функція | Доступ | Де викликати | Дія |
|---|---|---|---|
| `GetHandCount()` | BP | Будь-де | Кількість рук, що тримають предмет |
| `GetPrimaryHand()` / `GetSecondaryHand()` | BP | Будь-де | Основна й підтримувальна руки |
| `GetManipulatedComponent()` | BP | Будь-де | Компонент, яким рухає маніпуляція |
| `ValidateSetup(OutReason)` | BP | Будь-де | Перевірити профіль і ціль; причина відмови в `OutReason` |
| `GetHandleValue()` | BP | Будь-де | Хід ручки: градуси для `Rotary`, сантиметри для `Linear` |
| `IsExternallyDriven()` | BP | Будь-де | Рухом керує зовнішній драйвер |
| `CancelAllGrips()` | BP, authority only | Сервер | Завершити всі хвати без кидка |
| `ResetManipulation()` | BP, authority only | Сервер | Скасувати хвати, драйвер і прикріплення; повернути вільний предмет у `HomeTransform`, ручку — у хід 0, іменовану частину — під її батька |
| `SetGrabEnabled(bEnabled)` | BP, authority only | Сервер | Дозволити чи заборонити захоплення; вимкнення відпускає поточні руки |
| `GetHomeTransform()` / `SetHomeTransform(Transform)` | BP; запис — authority only | Будь-де / сервер | Домашня поза для `ResetManipulation`; запам'ятовується в `BeginPlay` |
| `SyncTransform()` | BP, authority only | Сервер | Негайно надіслати клієнтам позу, змінену gameplay-кодом |
| `SetExternalDriver(Driver)` | BP, authority only | Сервер | Передати рух іншому актору; `None` повертає звичайну маніпуляцію |
| `SetExternalPose(Pose)` | BP, authority only | Сервер | Поза предмета під зовнішнім драйвером |
| `GetSettings()` | C++ | Будь-де | Ефективні налаштування (на клієнті — з snapshot) |
| `AttachToSurface(Parent, Socket)` | BP, authority only | Сервер | Прикріпити вільний предмет до компонента, зберігши світову позу |
| `SetFreeSimulation(bSimulate, Velocity, AngularVelocity)` | BP, authority only | Сервер | Для вільного предмета: зняти прикріплення й драйвер, увімкнути чи вимкнути симуляцію зі швидкостями |
| `IsAttached()` | BP | Будь-де | Предмет прикріплено до поверхні |

| Подія | Де спрацьовує | Параметри |
|---|---|---|
| `OnGripStarted` | Сервер | `Hand`, `bFirst` — `true`, якщо предмет до цього був вільним |
| `OnGripEnded` | Сервер | `Hand`, `bLast`, `Reason`, `LinearVelocity`, `AngularVelocity` |
| `OnUsed` | Сервер | `Hand` — рука, що тримає предмет і викликала `RequestInteraction` |
| `OnHandleChanged` | Сервер | `Value`, `Fraction` 0..1; також `(0, 0)` після `ResetManipulation` |

### FInteractionManipulationSettings

| Поле | Початково | Призначення |
|---|---|---|
| `Movement` | `Direct` | `Direct` — точна поза щокадру; `Physics` — серверний `PhysicsHandle` із колізіями |
| `GripSelection` | `Free` | `Free`, `Authored` — лише точки хвату, `PreferAuthored` — точка, якщо поруч, інакше вільно |
| `Alignment` | `KeepOffset` | `KeepOffset` — без стрибка; `Align` — сумістити точку (або корінь) з долонею |
| `MaximumHands` | 2 | 1 або 2 руки одного pawn |
| `TwoHandPivot` | `Primary` | Опора основної руки або `Midpoint` між руками |
| `bAllowScaling` | `false` | Рівномірний масштаб двома руками (лише `Direct`, дві руки, без обмеження) |
| `MinimumScale` / `MaximumScale` | 0.25 / 4 | Межі масштабу за найбільшою віссю |
| `bSimulateOnRelease` | `true` | Увімкнути фізику після останньої руки |
| `bAllowThrow` | `true` | Передати швидкості при навмисному відпусканні |
| `LinearStiffness` / `LinearDamping` | 3000 / 200 | Привід позиції для `Physics` |
| `AngularStiffness` / `AngularDamping` | 2500 / 200 | Привід обертання для `Physics` |
| `BreakDistance` | 100 | На скільки тіло може відстати від бажаної пози понад своє найближче положення після захоплення; далі хват скасовується |
| `Constraint` | `None` | `Rotary` або `Linear` (лише `Direct`, одна рука, без масштабу) |
| `ConstraintAxis` | (1, 0, 0) | Вісь у початковій системі координат цілі |
| `MinimumValue` / `MaximumValue` | 0 / 720 | Межі ходу: градуси або сантиметри |

Готові нащадки змінюють частину значень: `AInteractionPhysicsActor` — `MaximumHands=1`, `AInteractionMasterItem` — `Alignment=Align`.

### UInteractionManipulationProfile

Data Asset (**Content Browser → Add → Miscellaneous → Data Asset → InteractionManipulationProfile**) з одним полем `Settings` типу `FInteractionManipulationSettings`. Профіль спільний для багатьох предметів і не змінюється під час гри.

### UInteractionGripPointComponent

| Властивість | Початково | Призначення |
|---|---|---|
| `Source` | `Component` | `Component` — трансформація самої точки; `Socket` — socket mesh |
| `SocketComponentName` | `None` | Компонент із socket; `None` — батьківський компонент точки |
| `SocketName` | `None` | Ім'я socket |
| `SocketOffset` | Identity | Корекція відносно socket |
| `bFallbackToComponent` | `false` | Дозволити трансформацію точки, якщо socket відсутній |
| `Radius` | 20 | Максимальна відстань долоні до точки |
| `AllowedHand` | `Any` | Обмеження сторони руки |
| `bAllowFirstHand` / `bAllowSecondHand` | `true` / `true` | Чи може точка почати хват / додати другу руку |
| `Priority` | 0 | Вищий пріоритет перемагає; серед рівних — найближча точка |

`ResolveAnchor(OutTransform)` (BP) повертає світову позу точки або `false`, якщо socket недоступний.

### UInteractionDockComponent

Посадкова опора на одну деталь. Сервер кожні 0.1 с шукає поруч вільну деталь.

| Властивість | Початково | Призначення |
|---|---|---|
| `RequiredPartTag` | `Interaction.Part` | Actor Tag, без якого деталь не сяде |
| `CaptureRadius` | 12 | Відстань від origin деталі до опори |
| `AlignmentTolerance` | 35° | Допустима похибка орієнтації |

Деталь сідає, якщо має `UInteractionManipulationComponent`, її ніхто не тримає й не резервує, вона не прикріплена й не під зовнішнім драйвером, а між нею й опорою немає перешкод на `Visibility`. Тоді опора вимикає симуляцію, ставить деталь у свою трансформацію (зберігаючи масштаб деталі) і прикріплює її. Повторне захоплення звільняє місце. `GetOccupant()` (BP) повертає деталь, `TryDock(Part)` (BP, сервер) пробує посадити її негайно, наприклад для початково зібраного стану.

### UInteractionBowComponent

Лук із пов'язаною стрілою: лук тримає одна рука, стрілу — інша рука того самого pawn.

| Властивість | Початково | Призначення |
|---|---|---|
| `ArrowTag` | `Interaction.Arrow` | Actor Tag стріли |
| `StringRestPoint` | `None` | Компонент або точка хвату з позою тятиви; `None` — `StringRestLocal` |
| `StringRestLocal` | (-15, 0, 0) | Поза тятиви відносно кореня лука; вісь X спрямована до цілі |
| `ArrowGuideLocal` | (15, 0, 0) | Точка, через яку проходить стріла |
| `ArrowNockPoint` | `None` | Компонент або точка хвоста стріли; `None` — корінь стріли |
| `NockRadius` | 18 | Відстань хвоста до тятиви для встановлення |
| `MaximumDraw` | 55 | Найбільший натяг |
| `MinimumShotFraction` | 0.1 | Менший натяг повертає тятиву без пострілу |
| `MaximumLateralOffset` | 35 | Бічне відхилення руки, що скасовує встановлення |
| `FullDrawSpeed` | 2200 | Швидкість при повному натязі |
| `DrawExponent` | 1 | Крива `Speed = FullDrawSpeed × Draw^DrawExponent` |

| Член | Доступ | Призначення |
|---|---|---|
| `GetDrawFraction()` | BP, реплікується | Натяг 0..1 для анімації тятиви |
| `GetNockedArrow()` | BP, реплікується | Встановлена стріла |
| `TryNock(Arrow)` | BP, сервер | Встановити утримувану сумісну стрілу |
| `CancelNock()` | BP, сервер | Зняти стрілу без пострілу |
| `OnShot` | Подія, сервер | `Projectile`, `Velocity` — після навмисного відпускання стріли |

Перед `OnShot` компонент сам вмикає симуляцію стріли з розрахованою швидкістю. Для `AInteractionSpearActor` викличте в обробнику `Launch(Velocity)`, щоб увімкнути політ і втикання.

### UInteractionStrikeComponent

Подія удару ребром утримуваного предмета; шкоду визначає гра.

| Властивість | Початково | Призначення |
|---|---|---|
| `EdgeStart` / `EdgeEnd` | (30, -5, 0) / (30, 5, 0) | Ребро в локальних координатах цілі маніпуляції |
| `EdgeRadius` | 2 | Товщина sweep |
| `MinimumSpeed` | 120 | Мінімальна швидкість точки ребра |
| `RequiredTargetTag` | `None` | Actor Tag цілі; `None` — будь-яка |

`OnStrike(Target, Hit, Speed)` спрацьовує на сервері один раз за епізод контакту, лише коли предмет тримають і ребро рухається назустріч поверхні. Швидкість рахується за час між двома позами предмета, тому кадри без нової пози віддаленої руки її не завищують. Потрібен `UInteractionManipulationComponent` на тому самому акторі.

### UInteractionDestructibleComponent

| Член | Доступ | Призначення |
|---|---|---|
| `Health` | Class Defaults | Запас міцності, 50 |
| `DamagePerImpulse` | Class Defaults | Шкода на одиницю імпульсу удару, 0.01 |
| `MinimumImpactDamage` | Class Defaults | Удар зараховується, якщо шкода більша за це значення (5) |
| `DestroyEffect` | Class Defaults | Ефект у точці руйнування: Niagara або Cascade |
| `GetHealth()` | BP | Залишок міцності на сервері |
| `ApplyImpactDamage(Amount, Location)` | BP, сервер | Завдати шкоди; при нулі — ефект і знищення власника |
| `OnImpactDestroyed(Location)` | Подія, сервер і всі клієнти | Перед знищенням власника: звук, уламки, власний ефект |

### UInteractionMeshComponent

`UStaticMeshComponent`, у якого вимкнено реплікацію трансформації та прикріплення. Його можна зробити replicated (mesh, матеріали, видимість), не створюючи другого «письменника» руху поряд зі snapshot маніпуляції. Звичайний нереплікований static mesh теж підходить.

### AInteractionPhysicsActor

Готовий replicated предмет: корінь `MeshComponent` (`UInteractionMeshComponent`), `Manipulation` і `IInteractableInterface`.

| Член | Доступ | Призначення |
|---|---|---|
| `ObjectData` | BP RO | [UInteractionPhysicsDataAsset](#uinteractionphysicsdataasset) |
| `bInteractionEnabled` | BP RW, реплікується | Вимикає і захоплення, і звичайну дію |
| `bCanBeGrabbed` | BP RW, реплікується | Дозвіл на захоплення; двері й кнопки — `false`. Змінюйте на сервері |
| `InteractionTags` | BP RW | Gameplay Tags для фільтра |
| `OnInteracted` | Подія, сервер | Успішна звичайна дія; параметр — interactor |
| `IsHeld()` | BP | Предмет тримає хоча б одна рука |
| `OnComponentHitInternal(...)` | C++, protected virtual | Реакція на вільний удар: звук і, якщо ввімкнено, шкода, див. [05](05-Physics.md#удари-звук-і-пошкодження) |
| `OnGrab` / `OnRelease` | Реалізація інтерфейсу | Запам'ятати ініціатора шкоди; запустити таймер `DisablePhysicsFixedTime`. У нащадку викликайте батьківську реалізацію |

`CanInteract` дозволяє `Grab`, якщо `bCanBeGrabbed`, і `Interact`, якщо предмет ніхто не тримає.

### AInteractionMasterItem

Фізичний предмет із локальною індикацією: `ProximitySphere` (350 см, overlap з Pawn), текстові маркери `NearbyMarker` («•») і `FocusMarker`, необов'язкові billboard `NearbyIcon` і `FocusIcon`, підсвітка mesh.

| Властивість | Початково | Призначення |
|---|---|---|
| `FocusOverlayMaterial` | `None` | Overlay-матеріал під час фокуса |
| `bUseCustomDepthFocus` | `true` | Писати mesh у custom depth для outline |
| `FocusStencil` | 252 | Значення stencil для вашого post-process |
| `InteractionPrompt` | `E` | Текст підказки фокуса |

`HasLocalFocus()` (BP) показує, чи фокусує предмет хоча б одна вільна локальна рука. Індикація оновлюється 10 разів на секунду й вимкнена на dedicated server.

### AInteractionSpearActor

| Член | Початково | Призначення |
|---|---|---|
| `PenetrationDepth` | 15 | Заглиблення вістря після влучання |
| `TipContactLength` | 40 | Довжина передньої частини mesh, що може встромитися |
| `MinimumEmbedSpeed` | 200 | Мінімальна швидкість для втикання |
| `Launch(Velocity)` | BP, сервер | Запустити політ без руки (стріла з лука) |
| `GetSpearTipLocation()` | BP | Світова позиція вістря (+X, `Bounds.Max.X`) |
| `IsEmbedded()` | BP | Спис устромлено в поверхню |

### UInteractionPhysicsDataAsset

| Поле | Початково | Призначення |
|---|---|---|
| `MassInKg` | 1 | Маса тіла |
| `bStartWithPhysics` | `false` | Симуляція з початку гри |
| `PostThrowBehavior` | `SleepWhenSettled` | `SleepWhenSettled` або `DisablePhysicsFixedTime` |
| `PhysicsDisableDelay` | 5 с | Затримка для `DisablePhysicsFixedTime` |
| `bAlignToVelocity` | `true` | Спис повертається вістрям за рухом |
| `AlignmentStiffness` | 5 | Швидкість цього повороту |
| `HitSound` | `None` | Звук удару |
| `HitSoundMinimumImpulse` | 150 | Найменший імпульс, що відтворює звук |
| `HitSoundCooldown` | 0.15 с | Найменший проміжок між звуками одного предмета |
| `bApplyImpactDamage` | `false` | Завдавати шкоди акторам, у які влучає вільний предмет |
| `DamagePerImpulse` | 0.005 | Шкода на одиницю імпульсу |
| `MinimumDamageImpulse` | 20 | Слабші удари шкоди не завдають |
| `DamageTypeClass` | `UDamageType` | Тип шкоди для `ApplyDamage` |

### Переліки World

| Перелік | Значення |
|---|---|
| `EInteractionMovement` | `Direct`, `Physics` |
| `EInteractionGripSelection` | `Free`, `Authored`, `PreferAuthored` |
| `EInteractionGripAlignment` | `KeepOffset`, `Align` |
| `EInteractionTwoHandPivot` | `Primary`, `Midpoint` |
| `EInteractionConstraint` | `None`, `Rotary`, `Linear` |
| `EInteractionAnchorSource` | `Component`, `Socket` |
| `EInteractionPostThrowBehavior` | `SleepWhenSettled`, `DisablePhysicsFixedTime` |

## Демо: InteractionSystemDemo

Класи демо — приклади використання публічного API. Їх можна переглядати й успадковувати, але не обов'язково постачати у своїй грі.

| Клас | Карта | Призначення |
|---|---|---|
| `AInteractionDemoCharacter`, `AInteractionDemoHUD`, `AInteractionDemoGameMode` | InteractionLab | Desktop-персонаж: E — взяти або використати, F — кинути; дії Enhanced Input створює в коді |
| `AInteractionDemoDoor` | InteractionLab | Двері на `AInteractionPhysicsActor` з `bCanBeGrabbed=false` |
| `AInteractionDemoSpear`, `AInteractionSpearDemoCharacter`, `AInteractionSpearDemoGameMode` | SpearRange | Спис із локальною індикацією та персонаж для метання |
| `AInteractionVRPawn`, `AInteractionVRHUD`, `AInteractionVRGameMode` | VRLab, VRLabBP | OpenXR-pawn із двома руками й desktop-емулятором |
| `AInteractionVRItem`, `AInteractionVRAxe`, `AInteractionVRBow`, `AInteractionVRArrow`, `AInteractionVREngine`, `AInteractionVRPanel` | VRLab | Предмети станцій |
| `AInteractionVRBunkerDoor`, `AInteractionVRChoppingTarget` | VRLab | Двері з колесом і поліно |
| `UInteractionVRPanelWidget` | VRLab, VRLabBP | Нативний віджет просторової панелі |
| `UInteractionVRBowVisualLibrary::StringSegmentTransform` | VRLab, VRLabBP | Поза сегмента тятиви за натягом |

`AInteractionVRPawn` має `bForceEmulation`, `InputContext`, `IsEmulating()`, `RequestLabReset()` і Blueprint-подію `OnLabResetRequested`, яку використовує VRLabBP.

## Налагодження: InteractionSystemDebug

Листовий модуль інструментів налагодження ([15](15-Debug-Tools.md)). У Shipping-збірках нічого не реєструє.

| Член | Доступ | Призначення |
|---|---|---|
| `UInteractionDebugLibrary::ToggleInteractionDebugOverlay(PlayerController)` | BP | Відкрити або закрити панель для локального гравця |
| `UInteractionDebugLibrary::IsInteractionDebugOverlayOpen(PlayerController)` | BP | Чи відкрита панель |
| `InteractionOverlay`, `InteractionStatus`, `InteractionLog`, `InteractionClaims`, `InteractionRelease`, `InteractionResetAll` | Консоль | Команди менеджера читів; опис — у [15](15-Debug-Tools.md#консольні-команди) |
