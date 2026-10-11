# 11 — Інтеграція в Blueprint

*[🇬🇧 English](../en/11-Blueprint-Integration.md) | 🇺🇦 Українська*

Розділ показує повний шлях без C++: персонаж, який бере й кидає предмети, предмет, двері, власна реакція на захоплення та дві VR-руки. Назви вузлів наведено так, як їх показує редактор Blueprint. Повний перелік властивостей — у [довіднику API](API-Reference.md).

## Перед початком

- Плагін увімкнено в **Edit → Plugins → InteractionSystem**, редактор перезапущено.
- Для мережевої гри pawn і всі цілі мають **Replicates**. Для одиночної гри це не обов'язково, але й не заважає.
- Колізія цілей блокує канал `Visibility` (або той `TraceChannel`, який ви оберете).

## Персонаж для desktop

### Компонент

1. Відкрийте Blueprint персонажа й у **Components → Add** додайте `InteractorComponent`. Перетягніть його на камеру, щоб він ішов за поглядом.
2. Залиште `Use Pawn Viewpoint` увімкненим: компонент сам бере точку огляду контролера.
3. За потреби змініть `Trace Distance` (300 см) і `Trace Radius` (0 — тонкий промінь; 5–10 — ширший sweep, яким легше влучити в дрібні предмети).
4. `Grab Offset` визначає, де триматиметься взятий предмет відносно камери. Значення `(70, 0, -15)` з InteractionLab тримає предмет перед гравцем трохи нижче центру екрана.

### Введення через Enhanced Input

1. Створіть **Content Browser → Add → Input → Input Action** `IA_Use` і `IA_Throw` з типом **Digital (bool)**.
2. Створіть **Input → Input Mapping Context** `IMC_Interaction`: `E` → `IA_Use`, `F` → `IA_Throw`.
3. У персонажі додайте подію **Event Receive Controller Changed**. Якщо `New Controller` є `PlayerController` і pawn `Is Locally Controlled`, викличте **Get Enhanced Input Local Player Subsystem → Add Mapping Context** з `IMC_Interaction`. Ця подія спрацює й тоді, коли контролер прийде на клієнт уже після `BeginPlay`.
4. Обробіть **IA_Use → Started**:
   - якщо `Get Held Actor` дійсний — викличте `Request Interaction`: предмет у руці отримає `On Used`;
   - інакше викличте `Refresh Focus`, візьміть `Get Focused Actor`, зробіть **Cast To InteractionPhysicsActor** і перевірте `Can Be Grabbed`; якщо так — `Request Grab`, інакше — `Request Interaction`.
5. Обробіть **IA_Throw → Started**: якщо `Get Held Actor` дійсний, викличте `Request Release` з `Linear Velocity = Get Forward Vector (камери) × 900` і `Angular Velocity = (0, 120, 0)`. Нульові вектори просто відпускають предмет без кидка.

Так працює `AInteractionDemoCharacter` з InteractionLab; відрізняється лише спосіб введення.

### Підказки в HUD

1. Виділіть `InteractorComponent` і в **Details → Events** натисніть **+** біля `On Focus Changed`. Подія спрацьовує лише на машині гравця й передає `Previous Actor` і `New Actor`. Показуйте підказку, коли `New Actor` дійсний, і ховайте її, коли він порожній.
2. Додайте `On Action Result`: `Target Actor`, `Action`, `Accepted`, `Reason`. Якщо `Accepted = false`, сервер відхилив дію, а `Reason` пояснює чому: `Claimed` — предмет у руках іншого гравця, `NotReachable` — не дотягнутися, `InvalidPose` — рука надто далеко чи за перешкодою тощо. Підключіть **Switch on EInteractionRejectReason** і покажіть коротке повідомлення; готова таблиця текстів — у [уроці 3](14-Tutorials.md#урок-3-пояснюємо-гравцеві-чому-дія-не-вдалася).
3. Поки рука тримає предмет або чекає відповіді, фокус порожній: `Can Focus New Target` повертає `false`. Для підказки «Кинути» перевіряйте `Get Held Actor`.

## Предмет, який можна взяти

### Варіант 1: готовий клас

1. **Content Browser → Add → Blueprint Class → All Classes** → `InteractionMasterItem` (з маркерами й підсвіткою) або `InteractionPhysicsActor` (без представлення).
2. Призначте `Mesh Component` static mesh із simple collision.
3. Створіть Data Asset від `InteractionPhysicsDataAsset` (**Add → Miscellaneous → Data Asset**) і задайте `Mass In Kg`, `Start With Physics`, `Hit Sound`. Призначте його в `Object Data`.
4. Для звичайного предмета налаштувань `Manipulation` досить. Для двох рук, масштабування чи фізичного утримання призначте `Manipulation → Profile` (див. [08](08-VR-Manipulation.md)).

Готовий фізичний актор відтворює `Hit Sound` від сильних ударів, а шкоди не завдає, доки ви не ввімкнете `Apply Impact Damage` в `Object Data` ([05](05-Physics.md#удари-звук-і-пошкодження)).

### Варіант 2: власний актор із компонентами

1. Створіть Blueprint Actor, увімкніть **Replicates**.
2. Кореневий компонент — Static Mesh із **Mobility = Movable** і simple collision. Не вмикайте для нього **Component Replicates**: рух передає маніпуляція. Якщо mesh має реплікуватися (матеріали, видимість), використайте як корінь `InteractionMeshComponent`.
3. Додайте `InteractionManipulationComponent`. Без профілю він використовує `Inline Settings`; для однієї руки задайте `Maximum Hands = 1`.
4. Задайте теги для фільтрів у `Target Tags` компонента.
5. Для точних місць хвату додайте `InteractionGripPointComponent` і виберіть `Grip Selection = Authored` або `Prefer Authored`.

Актор без `IInteractableInterface` не отримує `On Focus Start/End` і не має звичайної дії, тому фокус на ньому з'являється лише тоді, коли його зараз можна взяти. Для підсвітки чи власної підказки реалізуйте інтерфейс.

## Двері або кнопка

1. Створіть Blueprint Actor з **Replicates**, mesh із колізією й **Mobility = Movable**.
2. **Class Settings → Interfaces → Add** → `InteractableInterface`.
3. Додайте змінну `bOpen` з **Replication = RepNotify**. У функції `OnRep_bOpen` поверніть стулку (Timeline або `Set Relative Rotation`).
4. Реалізуйте **Event Interaction**: `Set bOpen = NOT bOpen`. Подія виконується на сервері, тому RepNotify донесе зміну до клієнтів. На сервері в Blueprint `OnRep` викликається автоматично після `Set` з RepNotify.
5. Перевизначте `Can Interact`: поверніть `true` для `Action = Interact` і `false` для `Grab`. Функція виконується й на клієнті для фокуса, тому вона не повинна нічого змінювати.
6. За бажанням реалізуйте `On Focus Start` і `On Focus End` для локальної підсвітки: вони виконуються лише на машині гравця, що навів руку.

Простіший шлях — дочірній клас `InteractionPhysicsActor` із `Can Be Grabbed = false`: прив'яжіться до `On Interacted` і змінюйте replicated-змінну так само. Головне правило: усе, що бачать інші гравці, змінюйте через реплікацію, а не напряму в `Interaction`.

## Реакція на взяття, використання й відпускання

На `InteractionManipulationComponent` предмета (**Details → Events**):

| Подія | Коли | Типове використання |
|---|---|---|
| `On Grip Started (Hand, First)` | Рука взяла предмет; `First = true` для першої руки | Звук підбору, вимкнення AI-поведінки предмета |
| `On Grip Ended (Hand, Last, Reason, Linear Velocity, Angular Velocity)` | Рука відпустила предмет | Звук, запуск таймера, повернення на місце; `Reason = Released` означає навмисне відпускання |
| `On Used (Hand)` | Рука, що тримає предмет, викликала `Request Interaction` | Постріл, вмикання ліхтаря, натискання кнопки на пульті |
| `On Handle Changed (Value, Fraction)` | Змінився хід обертової чи лінійної ручки | Відкрити двері, коли `Fraction` досягне 1 |

Усі ці події спрацьовують **лише на сервері**. Візуальні наслідки передавайте клієнтам через replicated-змінні або multicast-події.

Актор з `InteractableInterface` додатково отримує події інтерфейсу `On Grab` (перша рука взяла) і `On Release` (остання рука відпустила, з `Reason` і швидкостями) — так само на сервері.

Готовий приклад — колесо й двері VRLabBP: `BP_BunkerWheel` слухає `On Handle Changed`, а `BP_BunkerDoor` після розблокування викликає `Cancel All Grips`, `Set External Driver` і щокадру `Set External Pose`, щоб колесо рухалося разом зі стулкою ([09](09-VRLabBP.md)).

## Серверні функції маніпуляції

Ці функції `InteractionManipulationComponent` доступні в Blueprint і виконуються лише на сервері (на клієнті вузол нічого не робить):

| Функція | Призначення |
|---|---|
| `Set Grab Enabled` | Заборонити або дозволити захоплення; вимкнення відпускає поточні руки, клієнти бачать нове значення |
| `Reset Manipulation` | Звільнити руки й повернути предмет додому ([08](08-VR-Manipulation.md#повернення-на-місце)) |
| `Set Home Transform` / `Get Home Transform` | Змінити або прочитати домашню позу |
| `Attach To Surface` / `Is Attached` | Прикріпити вільний предмет до компонента іншого актора; повторне захоплення знімає прикріплення |
| `Set Free Simulation` | Зняти прикріплення й увімкнути чи вимкнути симуляцію, за потреби з початковою швидкістю |
| `Set External Driver` / `Set External Pose` | Передати рух іншому актору, як колесо на дверях |
| `Cancel All Grips` | Звільнити всі руки без кидка |
| `Sync Transform` | Негайно надіслати клієнтам позу, змінену вашим кодом |

Мережеві межі джерел — відстань, інтервал, швидкості кидка — налаштовуються в **Project Settings → Plugins → Interaction System** і перевизначаються на окремому `InteractorComponent` ([04](04-Multiplayer-and-VR.md#мережеві-межі)).

## Власна ціль захоплення в Blueprint

Якщо предмет має поводитися в руці інакше, ніж уміє маніпуляція, створіть **Blueprint Class → All Classes → InteractionBlueprintGrabTargetComponent** і перевизначте його функції:

| Функція | Де виконується | Що робити |
|---|---|---|
| `Can Begin Grip` | Клієнт (для підказки) і сервер | Повернути `false`, щоб заборонити хват; читати лише реплікований стан |
| `Begin Grip` | Сервер | Прикріпити чи рухати власника; `false` скасовує хват (клієнт отримає `Unavailable`) |
| `End Grip` | Сервер | Відпустити власника; `Reason = Released` — навмисне відпускання |
| `Use` | Сервер | Дія предмета в руці; повернути `true`, якщо оброблено |

Компонент сам перевіряє `Grab Enabled`, вільне місце (`Maximum Hands`) і те, що друга рука належить тому самому гравцю. `Get Hands` і `Get Hand Count` реплікуються; `Set Grab Enabled` і `Cancel All Grips` викликаються на сервері. Повний приклад — ліхтар з [уроку 5](14-Tutorials.md#урок-5-ліхтар-на-blueprint-власна-ціль-захоплення-без-c).

## Дві VR-руки

1. У pawn додайте два `MotionControllerComponent` (`Motion Source`: `Left` і `Right`) під спільним scene component на рівні підлоги.
2. Під кожен контролер додайте `InteractorComponent`: `Use Pawn Viewpoint = false`, `Hand = Left/Right`, `Near Interaction = true`, `Grab Offset = (0, 0, 0)`. За потреби відкалібруйте `Palm Offset`.
3. У **Event Tick** для кожної руки:
   - `Set Tracking Valid(Is Tracked)` контролера;
   - запам'ятайте `Get Grip Transform → Location` і обчисліть швидкість як різницю з попереднім кадром, поділену на `Delta Seconds`. Згладжуйте її (`VInterp To`) — це швидкість кидка.
4. Grip натиснуто → `Request Grab`; відпущено → `Request Release(швидкість, кутова швидкість)`.
5. Trigger → `Request Interaction`: використання предмета в руці або звичайна дія вільною рукою.
6. Перед телепортом чи recenter викличте `Cancel Grip` з `Reason = Recentered` для обох рук.

Готовий pawn із цим усім і desktop-емулятором — `InteractionVRPawn`; його можна взяти батьківським класом, як `BP_VRLabPawn` у VRLabBP.

## Часті помилки

| Симптом | Причина |
|---|---|
| Фокус є, але сервер відхиляє дію | Подивіться `Reason` в `On Action Result` або панель `InteractionOverlay` ([15](15-Debug-Tools.md)): ціль без **Replicates** (`InvalidTarget`), рука далі за `Max Origin Distance` чи за перешкодою (`InvalidPose`), ціль у чужих руках (`Claimed`) |
| Предмет не вибирається | Колізія не блокує `Visibility`; актор не має ні інтерфейсу, ні компонента цілі; спрацював `Filter` |
| Двері відчиняються лише в хоста | Стулку повертають прямо в `Interaction` без реплікації |
| Предмет «смикається» у клієнтів | На акторі ввімкнено `Replicate Movement` або реплікацію кореневого компонента поряд з маніпуляцією |
| VR-рука шукає предмети від камери, а не від долоні | У компонента руки залишився `Use Pawn Viewpoint = true` |
| Після телепорту предмет відлітає | Перед переміщенням не викликали `Cancel Grip(Recentered)` |

Більше відповідей — у [FAQ](FAQ.md) та [обмеженнях](Limitations.md).
