# Нотатки до випусків

*[🇬🇧 English](../en/Release-Notes.md) | 🇺🇦 Українська*

Коротко про кожен випуск: що з'явилося, що виправлено, на що звернути увагу під час оновлення. Найновіший випуск — угорі.

## 1.0

Unreal Engine **5.8**. Перший випуск плагіна. Попередній прототип має інший API; шару сумісності з ним немає, тож переносьте інтеграцію за [швидким стартом](02-Quick-Start.md).

**Що всередині:** [промінь, sphere sweep, фокус і теги](03-Interaction-and-Focus.md); [серверні запити, реєстр захоплень та дві VR-руки](04-Multiplayer-and-VR.md); [фізичні предмети, спис і руйнування](05-Physics.md); [VR-маніпуляції з профілями](08-VR-Manipulation.md): дві руки одного гравця, socket/component/free anchors, масштабування й обертання, фізичне утримання, лук зі стрілою, маховик, посадка деталі та події удару.

**Мережа:**

- Сервер перевіряє позу кожної руки окремо: перешкода перед вільною рукою більше не змушує сервер упустити предмет в іншій руці, а локально вимкнена перша рука не зупиняє оновлення поз.
- `OnActionResult` повідомляє причину відмови `EInteractionRejectReason`: `Cooldown`, `InvalidTarget`, `InvalidPose`, `NotReachable`, `HandBusy`, `Claimed`, `Unavailable` ([13](13-Network-Model.md#причини-відмови)).
- Мережеві межі задаються в **Project Settings → Plugins → Interaction System** і перевизначаються на окремому `InteractorComponent` ([04](04-Multiplayer-and-VR.md#мережеві-межі)).
- `bGrabEnabled` і `bCanBeGrabbed` реплікуються; `SetGrabEnabled` вимикає захоплення й безпечно відпускає поточні руки.
- Фізичний предмет у руці власника передбачається й не відстає на час обміну з сервером.
- Знімок маніпуляції квантується й підтримує push model; нерухомі предмети тікають раз на 0.5 с і прокидаються від руху ([Продуктивність](Performance.md)).

**Маніпуляція:**

- Іменована частина актора (`TargetComponentName`) — шухляда, кришка, важіль — залишається під своїм батьком, рухається разом із ним і повертається на місце після скидання.
- `ResetManipulation` повертає предмет у домашню позу з початку гри; `SetHomeTransform` її змінює.
- `BreakDistance` рахується від найближчого підходу тіла, тож фізичний предмет, узятий здалеку, не відпускається одразу.
- `AttachToSurface`, `SetFreeSimulation`, `IsAttached`, `SyncTransform` і запити реєстру захоплень доступні в Blueprint.

**Цілі:**

- `UInteractionBlueprintGrabTargetComponent` дозволяє описати власну ціль захоплення в Blueprint ([11](11-Blueprint-Integration.md#власна-ціль-захоплення-в-blueprint)).
- Події інтерфейсу `OnGrab(Interactor)` і `OnRelease(Interactor, Reason, LinearVelocity, AngularVelocity)` надходять від ядра для будь-якої цілі; Blueprint може їх реалізувати, але не викликати.
- Актор лише з компонентом цілі отримує фокус тільки тоді, коли його зараз можна взяти.

**Фізика й удари:**

- Шкода від ударів фізичного актора вимкнена за замовчуванням і налаштовується в `ObjectData`; звук удару має поріг і паузу між повтореннями ([05](05-Physics.md#удари-звук-і-пошкодження)).
- `UInteractionDestructibleComponent` приймає ефекти Niagara або Cascade, має налаштовувані пороги й подію `OnImpactDestroyed`.
- Швидкість удару `UInteractionStrikeComponent` рахується за час між позами предмета.

**Інструменти:** модуль `InteractionSystemDebug` з панеллю на екрані й командами `InteractionOverlay`, `InteractionStatus`, `InteractionLog`, `InteractionClaims`, `InteractionRelease`, `InteractionResetAll` ([15](15-Debug-Tools.md)). У Shipping-збірках модуль нічого не реєструє.

**Демо:** InteractionLab і [SpearRange](07-Spear-Range.md) з п'ятьма списами, мішенями, відмітками відстані й кнопками E/F; карта VRLab із сімома станціями й desktop-емулятором; [Blueprint-карта VRLabBP](09-VRLabBP.md) з тими самими механіками на Event Graphs. Desktop-персонаж використовує Enhanced Input, створений у коді. Розколоте поліно у VRLabBP реплікується всім гравцям. VRLab і VRLabBP використовують спільну палітру з `/InteractionSystem/Shared/Materials`. Увесь контент — усередині плагіна; модулі Demo і Debug можна видалити, а тести пропускають їхні перевірки.

**Документація:** [покрокові уроки](14-Tutorials.md), [словник термінів](Glossary.md), інтеграція в [Blueprint](11-Blueprint-Integration.md) і [C++](12-Cpp-Integration.md), [мережева модель](13-Network-Model.md), [довідник API](API-Reference.md) і сторінка [обмежень](Limitations.md).

Окремого CHANGELOG.md немає — зміни записуються на цій сторінці.
