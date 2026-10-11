# Архітектура

*[🇬🇧 English](../en/Architecture.md) | 🇺🇦 Українська*

## Модулі

| Модуль | Тип | Відповідальність | Залежності |
|---|---|---|---|
| InteractionSystem | Runtime | Контракти цілі, правила фільтрації, джерело променя, мережеві запити, реєстр захоплень, налаштування проєкту, Blueprint-ціль захоплення, математика хвату | Core, CoreUObject, Engine, GameplayTags, DeveloperSettings |
| InteractionSystemWorld | Runtime | Маніпуляція й профілі, точки хвату, лук, посадкова опора, удари, руйнування, фізичні актори, master item, спис | Core, CoreUObject, Engine, GameplayTags, InteractionSystem; приватно NetCore, Niagara |
| InteractionSystemDemo | Runtime | InteractionLab, SpearRange, VR Lab, Quest 2/OpenXR pawn, HUD, двері та приклади предметів | Core, CoreUObject, Engine, InputCore, InteractionSystem, InteractionSystemWorld, HeadMountedDisplay, XRBase, EnhancedInput, UMG, Slate, SlateCore |
| InteractionSystemDebug | Runtime | Консольні команди, журнал відповідей сервера й панель налагодження; у Shipping нічого не реєструє | Core, CoreUObject, Engine, InteractionSystem; приватно InteractionSystemWorld, InputCore, UMG, Slate, SlateCore |
| InteractionSystemTests | Editor | Поведінкові та PIE network тести через публічний API | Core, CoreUObject, Engine, GameplayTags, InputCore, UMG, UnrealEd, InteractionSystem, InteractionSystemWorld |

```text
InteractionSystem  <-  InteractionSystemWorld  <-  InteractionSystemDemo
        ^                        ^             <-  InteractionSystemDebug
        +---- InteractionSystemTests (лише публічний API; Demo і Debug — за іменем, без залежності)
```

Ядро не імпортує World, UI-бібліотеки або editor-модулі. Рендеринг та фізичні реалізації цілей знаходяться зовні контракту. Demo і Debug — листові модулі: від них ніхто не залежить, тому їх можна видалити ([06](06-Demo.md#виключення-з-packaged-game), [15](15-Debug-Tools.md#як-прибрати-модуль)); тести звертаються до них лише за іменами класів і пропускають свої перевірки, якщо модуля немає. Чисті правила `FInteractionFilter`, `FInteractionRules` і `FInteractionGripMath` не звертаються до світу чи RPC; UE scene/trace/network adapter — `UInteractorComponent`. Плагін вмикає залежні плагіни `EnhancedInput` і `XRBase`.

Немає окремого use case класу для кожного пересилання виклику: протокольні межі розміщені там, де змінюється відповідальність. Створення компонентів у pawn/Blueprint та налаштування цілей — composition root конкретної гри.

## Ролі класів

| Роль | Клас | Модуль |
|---|---|---|
| Джерело дії: фокус, запити, пози однієї руки чи камери | `UInteractorComponent` | Core |
| Контракт звичайної дії й локального фокуса | `IInteractableInterface` | Core |
| Контракт цілі захоплення | `UInteractionGrabTargetComponent`; Blueprint-варіант `UInteractionBlueprintGrabTargetComponent` | Core |
| Мережеві межі проєкту | `UInteractionSystemSettings` | Core |
| Серверний реєстр права на ціль | `UInteractionClaimSubsystem` | Core |
| Правила й математика без світу | `FInteractionFilter`, `FInteractionRules`, `FInteractionGripMath` | Core |
| Виконавець руху та replicated snapshot предмета | `UInteractionManipulationComponent`, `UInteractionManipulationProfile`, `UInteractionGripPointComponent` | World |
| Пов'язані дії | `UInteractionBowComponent`, `UInteractionDockComponent` | World |
| Gameplay-події без рішення про шкоду | `UInteractionStrikeComponent` | World |
| Прості gameplay-реакції | `UInteractionDestructibleComponent`, шкода від ударів `AInteractionPhysicsActor` | World |
| Готові актори | `AInteractionPhysicsActor`, `AInteractionMasterItem`, `AInteractionSpearActor` | World |
| Налагодження | `UInteractionCheatManagerExtension`, `UInteractionDebugLibrary`, журнал відповідей і панель | Debug |

## Потоки даних

### Фокус (лише локально)

```text
Tick interactor
  -> UpdateViewpoint (desktop) або поза батьківського motion controller (VR)
  -> RefreshFocus з частотою FocusUpdateHz
       -> TraceTarget(Grab); якщо ціль не знайдено -> TraceTarget(Interact)
  -> SetFocus: OnFocusEnd(старої) -> OnFocusStart(нової) -> OnFocusChanged
```

Фокус не реплікується й ніколи не є доказом дозволу: сервер повторює перевірки сам.

### Дія

```text
Input -> RequestGrab / RequestInteraction / RequestRelease
  -> CapturePoses (пакет поз усіх рук pawn)
  -> Server RPC на interactor pawn
  -> ExecuteAction: інтервал, AcceptPoses, повторний TraceTarget, CanBeginGrip, TryClaim
  -> BeginGrip | Interaction | Use | EndGrip
  -> HeldGrip (реплікація) + ClientActionResult -> OnActionResult
```

### Пози й рух предмета

```text
Клієнт: перший активний interactor -> ServerUpdatePoses (PoseReplicationHz), номер зразка в кожної руки свій
Сервер: AcceptPoses -> перевірка кожного зразка окремо -> серверні копії рук
Сервер: Tick маніпуляції (після рук) -> SolveDesired -> SetWorldTransform або PhysicsHandle
       -> State (квантована поза, push model) -> реплікація
Клієнти: плавне наближення до State; власник розв'язує позу сам (Direct) або передбачає (Physics)
Спокій: тік раз на IdleUpdateInterval; рух чи пробудження тіла -> знову щокадру
```

### Пов'язані дії

Лук резервує стрілу через `ReserveLinked`, стає її зовнішнім драйвером (`SetExternalDriver`) і щотіку задає позу (`SetExternalPose`). Навмисне відпускання стріли обчислює швидкість на сервері, вмикає симуляцію й викликає `OnShot`. Посадкова опора прикріплює вільну деталь через `AttachToSurface`; повторне захоплення знімає прикріплення.

## Інваріанти

- **Один реєстр права.** Лише `UInteractionClaimSubsystem` вирішує, хто тримає предмет. World-компоненти не заводять власних реєстрів: маніпуляція, лук і опора використовують один граф claims. Зовнішні системи можуть читати claimant, але не повинні змінювати ownership, обходячи `TryClaim`/`ReleaseClaim`.
- **Один «письменник» руху.** Позу, масштаб, прикріплення й симуляцію предмета змінює лише його маніпуляція. Тому вона вимикає `ReplicateMovement` власника, вимагає нереплікованого (або `UInteractionMeshComponent`) кореня, а `AInteractionPhysicsActor` ігнорує `OnRep_ReplicatedMovement` і `OnRep_AttachmentReplication`.
- **Сервер вирішує gameplay.** Клієнт показує фокус, але право на дію, фізика, шкода й постріл визначаються на сервері.
- **Кидок лише навмисний.** Лише причина `Released` передає швидкості. Втрата трекінгу, скасування, recenter і невалідна ціль відпускають предмет без кидка; лук без навмисного відпускання не стріляє.
- **Ідентичність запиту.** `HeldGrip` зберігає ціль разом із `RequestId`, тому запізнілі відповіді не скасовують нові запити.
- **Слабкі посилання.** Реєстр і локальні набори фокуса не продовжують життя акторів і компонентів.

## Точки розширення

| Потрібно | Як |
|---|---|
| Двері, кнопка, перемикач | `IInteractableInterface` у Blueprint або C++ ([11](11-Blueprint-Integration.md), [12](12-Cpp-Integration.md)) |
| Предмет, який беруть руками | `UInteractionManipulationComponent` на будь-якому акторі або нащадок `AInteractionPhysicsActor` |
| Інша поведінка утримання | Blueprint-нащадок `UInteractionBlueprintGrabTargetComponent` або C++-нащадок `UInteractionGrabTargetComponent` |
| Повідомлення гравцеві про відмову | `OnActionResult` з `EInteractionRejectReason` |
| Реакція на взяття, використання, відпускання, хід ручки | Події маніпуляції `OnGripStarted`, `OnGripEnded`, `OnUsed`, `OnHandleChanged` |
| Предмет рухається разом з іншим актором | `SetExternalDriver` + `SetExternalPose` |
| Власна індикація | `OnFocusChanged` interactor або `OnFocusStart`/`OnFocusEnd` цілі; `AInteractionMasterItem` — приклад |
| Власна реакція на удар фізичного актора | Перевизначення `OnComponentHitInternal` у C++ |
| Удар ребром | `UInteractionStrikeComponent::OnStrike` |

## Порядок оновлення

Усе виконується в ігровому потоці, без асинхронних задач.

| Хто | Коли тікає |
|---|---|
| `UInteractorComponent` | `TG_PrePhysics`, щокадру, коли активний |
| `UInteractionManipulationComponent` | `TG_PrePhysics`; щокадру, поки є робота, інакше раз на `IdleUpdateInterval` (0.5 с); під час хвату — після рук, що його тримають |
| `UInteractionBowComponent` | `TG_PrePhysics`, після маніпуляції лука; пошук стріли — раз на 0.05 с |
| `UInteractionStrikeComponent` | `TG_PostPhysics`, лише на сервері, коли предмет тримають; кадри без руху предмета пропускаються |
| `UInteractionDockComponent` | Раз на 0.1 с на сервері |
| `AInteractionMasterItem` | Раз на 0.1 с, не на dedicated server |
| `AInteractionSpearActor` | Лише під час польоту, на сервері |

## Що не входить до плагіна

Локомоція й телепорт, input mappings вашої гри (демо має власні), інвентар, анімація чи IK рук, UI-віджети (крім демо), модель шкоди й здоров'я (крім простих реакцій World), збереження стану. Ці частини лишаються за грою: плагін надає точки розширення вище.
