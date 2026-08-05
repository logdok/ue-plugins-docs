# 08 — Довідник Blueprint-бібліотеки

*[🇬🇧 English](../en/08-Blueprint-Library-Reference.md) | 🇺🇦 Українська*

`UQuestBlueprintLibrary` — рекомендована точка входу майже для всього: кожна функція `static`, викликається і з Blueprint, і з C++, і приймає `APlayerState*` (або world context), щоб самостійно маршрутизувати до потрібного менеджера (`PlayerState` чи `GameState`) — шукати `UQuestManagerComponent` вручну доводиться рідко.

Усі функції живуть у `Core/QuestBlueprintLibrary.h`.

## Доступ до менеджера

| Функція | Повертає |
|---|---|
| `GetQuestManager(Player)` | Особистий менеджер гравця (Personal + Individual квести). |
| `GetQuestManagerForController(PlayerController)` | Те саме, знайдене через контролер. |
| `GetPartyQuestManager(WorldContextObject)` | Менеджер `GameState` — колективний прогрес для Shared-квестів, *і* ростер учасників / відстеження особистого завершення для Individual-квестів (реальний прогрес цілей Individual-квесту, як і раніше, лежить на власному менеджері кожного гравця). |
| `GetQuestManagerForQuest(Player, Quest)` | **Те, що треба брати, якщо режим шарингу заздалегідь невідомий.** Повертає менеджер, якому реально належить цей квест: особистий — для Personal/Individual, party — для Shared. |

⚠️ Узяти `GetQuestManager()` для **Shared**-квесту — найчастіша помилка під час роботи з плагіном: повернеться менеджер гравця, у якому Shared-прогресу немає ніколи, і квест мовчки прочитається як «недоступний / не активний / не завершений», без жодної помилки будь-де. Використовуйте `GetQuestManagerForQuest()` усюди, де квест може виявитися Shared, або перевіряйте обидва менеджери, якщо збираєте список.

## Керування квестами

| Функція | Призначення |
|---|---|
| `StartQuest(Player, Quest)` | Запустити Personal-квест. Попереджає й завершується невдачею, якщо передано Shared-квест. |
| `StartPartyQuest(WorldContextObject, Quest, Participants)` | Запустити Shared/Individual квест із явним ростером. |
| `StartPartyQuestForAllPlayers(WorldContextObject, Quest)` | Запустити Shared/Individual квест на всіх поточних підключених гравцях — див. [06](06-Multiplayer.md). |
| `AddPartyQuestParticipant` / `RemovePartyQuestParticipant` | Керування ростером party-квесту. Авторитет сервера, безпечно викликати з будь-якого клієнта. Видалення останнього учасника автоматично провалює Shared-квест. |
| `CompleteQuest(Player, Quest)` | Вручну завершити квест, готовий до здачі. Маршрутизує за режимом розподілу прогресу автоматично. |
| `FailQuest(Player, Quest)` | Провалити квест. |
| `AbandonQuest(Player, Quest)` | Скасування з ініціативи гравця (потребує `bCanAbandon`). |

Усе перелічене вище (а також `StartPartyQuest`/`StartPartyQuestForAllPlayers`/`AddPartyQuestParticipant`/`RemovePartyQuestParticipant`) має авторитет сервера й безпечне для виклику з будь-якого клієнта, зокрема для Shared-квесту — див. [06 — Мультиплеєр](06-Multiplayer.md) (розділ «Авторитет сервера») про те, як це влаштовано, і, що важливіше, про застереження щодо значення, яке повертається, перш ніж будувати на ньому UI-логіку.

## Сповіщення про події

Повні правила зіставлення — у [04 — Подієвий прогрес](04-Event-Driven-Progress.md).

| Функція | Сигнатура |
|---|---|
| `NotifyQuestEvent` | `(Player, EventID, TargetIdentifier, Count = 1)` |
| `NotifyKillEvent` | `(Player, EnemyTag, Count = 1)` |
| `NotifyCollectEvent` | `(Player, ItemID, Count = 1)` |
| `NotifyInteractionEvent` | `(Player, TargetID)` |
| `NotifyLocationReachedEvent` | `(Player, LocationID)` |
| `NotifyCustomQuestEvent` | `(Player, EventID, TargetIdentifier, Count = 1)` |

## Запити щодо квестів

| Функція | Повертає |
|---|---|
| `GetAvailableQuests(Player, AllQuests)` | Квести з `AllQuests`, які гравець може почати просто зараз. |
| `IsQuestActive(Player, Quest)` / `IsQuestCompleted(Player, Quest)` | Перевірки статусу, маршрутизуються за режимом розподілу. |
| `CanStartQuest(Player, Quest)` | Пререквізити виконані + квест ще не активний/завершений/провалений (провалений квест спершу потребує `QuestReset`). |
| `IsTargetRelevant(Player, EventID, TargetID)` | `true`, якщо у гравця є активна ціль, яку просунув би `NotifyQuestEvent(EventID, TargetID)` (перевіряє свій + party менеджер). Керує маркерами world-цілей. |
| `GetActiveQuests(Player)` | Активні Personal + Individual квести гравця. |
| `GetActiveSharedQuests(WorldContextObject)` | Активні Shared-квести з `GameState`. |
| `GetCompletedQuests(Player)` | Лише завершені Personal + Individual квести гравця — завершений Shared-квест натомість лежить на `GameState`, а не тут. |
| `GetFailedQuests(Player)` | Лише провалені Personal + Individual квести гравця (вичерпався таймер, провалені вручну тощо) — те саме застереження щодо `GameState`, що й у `GetCompletedQuests`. |
| `GetQuestProgress(Player, Quest, OutObjectives, OutProgress)` | Паралельні масиви всіх цілей та їхнього поточного прогресу. |
| `GetQuestCompletionPercentage(Player, Quest)` | `0.0`–`1.0`, лише за обов'язковими цілями. Працює і для активного квесту, і для того, що вже завершився чи провалився (читає збережений знімок) — `0.0` означає лише «не розпочато або невідомо цьому менеджеру». |
| `AreAllRequiredObjectivesComplete(Player, Quest)` | `true`, щойно всі обов'язкові цілі стали Completed — як і відсоток вище, працює і після завершення квесту. **Не** те саме, що `IsQuestReadyToTurnIn` (використовується всередині для ручної здачі): та перевірка навмисно обмежена лише активними квестами, бо вже зданий квест більше не «готовий до здачі». |
| `GetQuestTimeRemaining(Player, Quest)` | Секунд лишилося у таймерного квесту (`0`, якщо не таймерний/не активний). |
| `GetQuestTimeRemainingText(Player, Quest)` | Те саме у форматі `MM:SS`. |
| `GetObjectiveTimeRemaining(Player, Quest, Objective)` | Секунд лишилося у таймерної *цілі* (`0`, якщо не таймерна/не активна) — незалежно від таймера самого квесту (див. [03](03-Authoring-Quests.md#таймерні-цілі)). |
| `GetObjectiveTimeRemainingText(Player, Quest, Objective)` | Те саме у форматі `MM:SS`. |

## Утиліти прогресу (без стану)

Корисні для UI-коду, у якого на руках лише `FQuestObjectiveProgress` або значення enum, без пошуку менеджера:

| Функція | Повертає |
|---|---|
| `GetObjectiveProgressText(Progress)` | Текст на кшталт `"3/5"`. |
| `IsObjectiveCompleted(Progress)` | `bool`. |
| `GetObjectiveProgressPercentage(Progress)` | `0.0`–`1.0`. |
| `GetQuestStateText(State)` | Готовий до локалізації текст для `EQuestState`. |
| `GetObjectiveStateText(State)` | Готовий до локалізації текст для `EQuestObjectiveState`. |

## Утиліти party-квестів

| Функція | Призначення |
|---|---|
| `GetPartyQuestParticipants(WorldContextObject, Quest)` | Ростер Shared або Individual квесту. |
| `GetPlayerContribution(WorldContextObject, Quest, Player)` | Частка гравця в прогресі Shared-квесту. |
| `IsPlayerParticipant(WorldContextObject, Quest, Player)` | Перевірка членства в ростері. |
| `HasPlayerCompletedIndividualQuest(Player, Quest)` | Перевірка особистого завершення Individual-квесту. |

## Куди далі

- Низькорівневий API `UQuestManagerComponent`, який огортають ці функції (делегати, нутрощі party-квестів): [01](01-Core-Concepts.md), [04](04-Event-Driven-Progress.md), [06](06-Multiplayer.md)
- Консольні команди, побудовані на тих самих запитах: [09 — Валідація та чити](09-Validation-And-Cheats.md)

<!-- doc-footer:start -->
---
*Last updated: 2026-08-05 11:23 UTC*
<!-- doc-footer:end -->
