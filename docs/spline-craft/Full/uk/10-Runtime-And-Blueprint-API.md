# Runtime, колізія, події, пресети, сценарії

*[🇬🇧 English](../en/10-Runtime-And-Blueprint-API.md) | 🇺🇦 Українська*

## Перебудова у грі

- **`UpdateAfterChangeAnyProperty()`** (`BlueprintCallable`) — викликати після зміни будь-яких
  властивостей структури в runtime. Працює **лише якщо Mobility ≠ Static**. Виконує
  вирівнювання точок, перебудову геометрії, повторний спавн додаткових акторів і прив'язку
  подій зіткнення.
- **Mobility** (`EComponentMobility`) — mobility кожного **згенерованого** компонента. Впливає
  на освітлення та участь у запіканні. Для runtime-перебудови задайте `Movable` або
  `Stationary`. Згенерований компонент ніколи не буває *статичнішим* за **Root Mobility** —
  задайте його `Movable`, і меші рухатимуться разом із коренем.
- **Root Mobility** (`EComponentMobility`, типово `Static`) — mobility власного кореня сплайна
  актора. Залиште `Static` для запеченого освітлення та поведінки, як до 7.x. Задайте
  `Movable`, щоб причепити компонент руху (напр. *Rotating Movement*) або рухати / обертати
  весь актор у runtime — тоді задайте `Movable` і для **Mobility**, щоб згенеровані меші
  рухалися з ним.
- **SplineComponent** — доступний лише для читання з Blueprint.

## Секція Stats (лише для читання)

Оновлюється після кожної збірки, у рівні не зберігається:

| Поле | Що показує |
|---|---|
| **Instanced Mesh Components** | скільки `InstancedStaticMeshComponent` дала остання збірка (по одному на унікальний меш у режимі Line). |
| **Total Instances** | скільки всього інстансів у всіх цих компонентах. |
| **Spline Mesh Components** | скільки `SplineMeshComponent` (по одному на елемент на проліт у режимі Curve — росте швидко). |
| **Polygon Components** | скільки процедурних мешів дали полігони. |

## Колізія (`FSCCollisionSettings`)

Секція **Collision** на акторі. Налаштування застосовуються **однаково до всіх** згенерованих
компонентів — інстансних мешів, сплайн-мешів і полігонів.

| Параметр | Призначення |
|---|---|
| **Override Collision** | увімкнути керування колізією. Поки вимкнено — стара поведінка: ISM залишаються на рушійному дефолті, а Spline Mesh примусово `Query Only`. |
| **Collision Profile** | профіль колізії для всіх компонентів. |
| **Generate Hit Events** | потрібно, щоб делегати `OnSplineCraft…HitEvent` спрацьовували взагалі. Також потрібен профіль із фізичною колізією (наприклад, `BlockAll`). |
| **Generate Overlap Events** | генерувати події перекриття. |
| **Polygons Create Collision** | будувати геометрію колізії для полігонів. Вимкніть для суто декоративних форм. |

> Інстансні компоненти діляться за ключем «меш + матеріали», тобто один компонент спільний
> для стовпів / секцій / труб / вузлів з однаковим мешем. Тому налаштування колізії глобальні
> на актор — розщепити їх по типу елемента без розщеплення ISM не можна.

## Події зіткнення

`FSplineCraftStaticMeshHitDelegate` (`BlueprintAssignable`), прив'язуються у `BeginPlay`:

- **OnSplineCraftSectionHitEvent**
- **OnSplineCraftTubeHitEvent**
- **OnSplineCraftPostHitEvent**
- **OnSplineCraftKnobHitEvent**

Параметри: `Sender`, `HitComponent`, `OtherActor`, `OtherComp`, `NormalImpulse`, `Hit`.

Щоб події працювали: **Override Collision** = увімк., **Generate Hit Events** = увімк.,
профіль колізії — з фізичним блокуванням (`BlockAll` чи схожий).

## Пресети (`USplineCraftPreset`)

Ассет-контейнер стилю. Поле **Preset** на акторі та дві кнопки в редакторі:

- **Apply Preset** — завантажити стиль на актор.
- **Save To Preset** — зберегти поточні налаштування актора в ассет.

Пресет несе: **Mode**, **Visible Last Point Elements**, **Posts**, **Tubes**, **Sections**,
**Knobs**, **Free Knobs**, **Polygons**, **Collision**, **Use Random Seed** / **Random Seed**,
**Mobility**.

Пресет **не торкається** того, що належить конкретному актору: форми сплайна, інструментів
розкладки, **Spline Close Loop**, **Openings** і **Design Notes**.

Створення: Content Browser → правий клік → *Miscellaneous → Data Asset* → клас
**SplineCraftPreset**. Готові приклади — у `Content/Samples`.

## Build Scenario — поетапна стройка

Ассет **SplineCraft Build Scenario** (`USplineCraftBuildScenario`) — упорядкована послідовність
пресетів із таймінгом. Актор із призначеним сценарієм стає **директором**: власної геометрії
не показує, а спавнить по дочірньому SplineCraft-актору на кожен крок — уздовж свого сплайна,
зі зсувом кроку. Так один пресет, піднятий по Z, дає поверхи будівлі. `Reverse` — та сама
послідовність на розбирання.

### Налаштування на акторі (секція Build Scenario)

| Параметр | Призначення |
|---|---|
| **Build Scenario** | посилання на ассет-сценарій. Порожнє — актор поводиться як звичайно. |
| **Play On Begin Play** | запустити сценарій автоматично на старті гри. |
| **Preview Progress** | *лише редактор*, 0→1: скрабіть — бачите стройку без таймерів. `0` очищає прев'ю. |

### Методи (Blueprint)

| Метод | Дія |
|---|---|
| **Play Build Scenario** (`bReverse`) | запустити / перезапустити. `false` — стройка, `true` — розбирання готової структури. |
| **Stop Build Scenario** | зупинити й видалити всі заспавнені кроки. |
| **Finish Build Scenario Now** | миттєво довести до кінцевого стану. |
| **Is Playing Build Scenario** | чи йде сценарій зараз. |

### Події

- **On Build Scenario Step Shown** (`StepIndex`, `Label`) — крок з'явився.
- **On Build Scenario Finished** — послідовність завершена.

### Ассет сценарію

`USplineCraftBuildScenario`: **Description**, **Start Delay** (с), **Loop** (+ **Loop Restart
Delay**, лише пряме програвання), **Steps**.

Крок (`FSCBuildStep`): **Label**, **Preset**, **Delay Before Show** (с), **Offset**
(`FTransform` відносно директора — піднімайте по Z для поверхів), **Appear Mode**,
**Appear Duration** (с), **Rise Distance** (см).

Режими появи (`ESCScenarioAppearMode`): **Instant**, **Rise From Below** (виїжджає знизу на
`Rise Distance`), **Scale Up** (росте від нуля), **Material Progress** (жене скалярний
параметр `SplineCraftBuildProgress` 0→1 на матеріалах кроку — працює, лише якщо матеріал
цей параметр читає).

Дочірні актори кроків — `Transient` (у рівень не зберігаються) і примусово `Movable`.
Повністю аддитивно: без призначеного сценарію поведінка актора не змінюється.

## Дивіться також

- [Продуктивність і запікання](13-Performance-And-Baking.md)
- [Додаткові актори](12-Additional-Actors.md)
