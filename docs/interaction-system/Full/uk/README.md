# InteractionSystem

*[🇬🇧 English](../en/README.md) | 🇺🇦 Українська*

Єдина взаємодія променем, серверне захоплення та локальний відгук для desktop, mobile і VR.

Версія [1.0](Release-Notes.md), Unreal Engine 5.8.

## Швидкий старт

1. Увімкніть **Edit → Plugins → InteractionSystem** у проєкті UE 5.8 і перезапустіть редактор.
2. Додайте `InteractorComponent` до replicated pawn і прикріпіть його до камери. Для desktop залиште `Use Pawn Viewpoint`; для VR додайте окремий компонент під кожну руку і вимкніть цей параметр.
3. Створіть Blueprint від `InteractionMasterItem`, призначте mesh із колізією, що блокує `Visibility`, і поставте його ближче ніж за 300 см від гравця.
4. Прив'яжіть клавішу до `Request Grab`, іншу — до `Request Release` зі швидкостями у см/с та градусах/с. Для дверей використайте `Request Interaction` і реалізуйте `Interaction`.
5. Натисніть **Play**: поруч з предметом з'явиться маркер, при наведенні — підказка `E`, після натискання предмет опиниться у вас у руці.
6. Якщо щось не працює, введіть у консолі `InteractionOverlay`: панель покаже фокус, стан руки й відповідь сервера ([15](15-Debug-Tools.md)).

Готовий приклад — карта `/InteractionSystem/Demo/InteractionLab/Maps/InteractionLab`: WASD і миша — рух, `E` — взяти або відчинити двері, `F` — кинути. Детальніше — у [02 — Швидкий старт](02-Quick-Start.md).

## Як читати цю документацію

| Ви… | Почніть з |
|---|---|
| Уперше робите взаємодію з предметами | [01 — Вступ](01-Introduction.md) → [02 — Швидкий старт](02-Quick-Start.md) → [14 — Покрокові уроки](14-Tutorials.md); незнайомі слова — у [словнику](Glossary.md) |
| Працюєте лише в Blueprint | [11 — Інтеграція в Blueprint](11-Blueprint-Integration.md) |
| Пишете на C++ | [12 — Інтеграція в C++](12-Cpp-Integration.md) і [довідник API](API-Reference.md) |
| Робите мультиплеєр або VR | [04 — Мультиплеєр та VR](04-Multiplayer-and-VR.md), [08 — VR-маніпуляції](08-VR-Manipulation.md), [13 — Мережева модель](13-Network-Model.md) |
| Шукаєте причину проблеми | [15 — Інструменти налагодження](15-Debug-Tools.md) і [FAQ](FAQ.md) |

## Зміст

- [Нотатки до випусків](Release-Notes.md)
- [01 — Вступ](01-Introduction.md)
- [02 — Швидкий старт](02-Quick-Start.md)
- [03 — Взаємодія та фокус](03-Interaction-and-Focus.md)
- [04 — Мультиплеєр та VR](04-Multiplayer-and-VR.md)
- [05 — Фізичні предмети](05-Physics.md)
- [06 — Демонстраційний контент](06-Demo.md)
- [07 — Метання списа](07-Spear-Range.md)
- [08 — VR-маніпуляції та профілі](08-VR-Manipulation.md)
- [09 — VRLabBP: демонстрація на Blueprint](09-VRLabBP.md)
- [10 — Як брати та використовувати предмети в демо](10-Demo-Controls.md)
- [11 — Інтеграція в Blueprint](11-Blueprint-Integration.md)
- [12 — Інтеграція в C++](12-Cpp-Integration.md)
- [13 — Мережева модель](13-Network-Model.md)
- [14 — Покрокові уроки](14-Tutorials.md)
- [15 — Інструменти налагодження](15-Debug-Tools.md)

## Довідка

- [Словник термінів](Glossary.md)
- [Довідник API](API-Reference.md)
- [Архітектура](Architecture.md)
- [Продуктивність](Performance.md)
- [Обмеження та відомі проблеми](Limitations.md)
- [Питання та відповіді](FAQ.md)
