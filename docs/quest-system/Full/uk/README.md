# Посібник користувача QuestSystem

*[🇬🇧 English](../en/README.md) | 🇺🇦 Українська*

Це повний довідковий посібник із плагіна QuestSystem — подієво-орієнтованої, дата-керованої системи квестів для **Unreal Engine 5.7** з повноцінною підтримкою мультиплеєра та інтеграцією без налаштування. Дизайнери створюють квести як Data Assets; геймплей-код повідомляє про кілька подій; решту робить плагін, включно з мережевою реплікацією.

## Швидкий старт

1. Скопіюйте `Plugins/QuestSystem` у папку `Plugins/` свого проєкту.
2. Увімкніть його: **Edit > Plugins > QuestSystem**.
3. Усе — код поки не потрібен. `UQuestWorldSubsystem` автоматично додає менеджер квестів кожному гравцю та ігровому стану.

Створіть свій перший квест:

1. У Content Browser: **Add > Miscellaneous > Data Asset**, оберіть **Quest Objective Data**. Задайте `ObjectiveType`, `TargetCount` та `ObjectiveIdentifier` (наприклад, `KillTarget` / `3` / `"Bandit"`).
2. Додайте ще один Data Asset, цього разу **Quest Data**. Дайте йому `QuestName` і додайте вашу ціль до `Objectives`.
3. Додайте **Quest Giver Component** на будь-який Actor на рівні та вкиньте свій квест у `AvailableQuests`.
4. Додайте **Quest Interactor Component** на пешку гравця та прив'яжіть введення до `TryInteract`.
5. Натисніть Play, підійдіть до актора, взаємодійте — квест почнеться. Повідомляйте про прогрес зі свого геймплей-коду:

```cpp
// напр., у коді обробки шкоди, коли ворог гине:
UQuestBlueprintLibrary::NotifyKillEvent(PlayerState, "Bandit");
```

Ціль оновиться, квест завершиться, і спрацює `GiveQuestRewards()` — перевизначте його (у Blueprint чи C++), щоб видавати власні нагороди. Решта цього посібника детально розкриває все вище, плюс мультиплеєр, збереження/завантаження, повний Blueprint API та точки розширення на C++.

## Кому що читати

- **Дизайнерам**, авторам контенту квестів: [01](01-Core-Concepts.md), [03](03-Authoring-Quests.md), [05](05-World-Components.md).
- **Геймплей-програмістам**, що підключають події та нагороди: [02](02-Zero-Config-Integration.md), [04](04-Event-Driven-Progress.md), [10](10-CPP-Integration.md).
- **Програмістам мультиплеєра**: [06](06-Multiplayer.md) після [01](01-Core-Concepts.md).
- **UI-програмістам**: [08](08-Blueprint-Library-Reference.md).
- **Тулінгу / QA**: [09](09-Validation-And-Cheats.md).

## Розділи

| # | Розділ | Що всередині |
|---|---------|---------------|
| 01 | [Основні поняття](01-Core-Concepts.md) | Квести, цілі, стани, режими розподілу прогресу, модель «один компонент — два місця» |
| 02 | [Інтеграція без налаштування](02-Zero-Config-Integration.md) | `UQuestWorldSubsystem`, `UQuestSystemSettings`, отримання менеджера квестів |
| 03 | [Авторинг квестів](03-Authoring-Quests.md) | Повний довідник полів `UQuestData` / `UQuestObjectiveData` |
| 04 | [Подієвий прогрес](04-Event-Driven-Progress.md) | `NotifyQuestEvent`, правила зіставлення, послідовні цілі, спільний залік убивств |
| 05 | [Компоненти сцени](05-World-Components.md) | Giver, Receiver, нащадки маркера, Interactor, Quest Target, Location Trigger, готові актори |
| 06 | [Мультиплеєр](06-Multiplayer.md) | Реплікація, party-квести, одиночна гра + мультиплеєр на одній карті |
| 07 | [Серіалізація](07-Serialization.md) | `ExportState` / `ImportState`, вбудовування у свій save game |
| 08 | [Довідник Blueprint-бібліотеки](08-Blueprint-Library-Reference.md) | Повний довідник функцій `UQuestBlueprintLibrary` |
| 09 | [Валідація та чити](09-Validation-And-Cheats.md) | Виявлення циклічних залежностей, консольні команди |
| 10 | [Інтеграція на C++](10-CPP-Integration.md) | Кастомні нагороди, розширення компонентів, надсилання подій із геймплей-коду |

## Домовленості цього посібника

- `UQuestData` — це ассет квесту; `UQuestObjectiveData` — ассет цілі. Обидва — `UPrimaryDataAsset`, створюються в Content Browser.
- Сніпети коду — на C++, якщо не вказано інше; кожна показана функція викликається і з Blueprint.
- «Менеджер» означає `UQuestManagerComponent` — де він живе і що робить, див. [01](01-Core-Concepts.md).

<!-- doc-footer:start -->
---
*Last updated: 2026-08-04 19:24 UTC*
<!-- doc-footer:end -->
