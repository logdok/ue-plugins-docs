# 08 — Довідник Blueprint-бібліотеки

*[🇬🇧 English](../en/08-Blueprint-Library-Reference.md) | 🇺🇦 Українська*

`UDemoQuestBlueprintLibrary` — рекомендована точка входу майже для всього: кожна функція `static`, викликається і з Blueprint, і з C++, і приймає `APlayerState*` (або world context), щоб самостійно маршрутизувати до потрібного менеджера (`PlayerState` чи `GameState`) — шукати `UDemoQuestManagerComponent` вручну доводиться рідко.

Усі функції живуть у `Core/DemoQuestBlueprintLibrary.h`.

## Доступ до менеджера

| Функція | Повертає |
|---|---|
| `GetQuestManager(Player)` | Особистий менеджер гравця (Personal + Individual квести). |
| `GetQuestManagerForController(PlayerController)` | Те саме, знайдене через контролер. |
| `GetPartyQuestManager(WorldContextObject)` | Менеджер `GameState` (Shared квести). |
| `GetQuestManagerForQuest(Player, Quest)` | **Те, що треба брати, якщо режим шарингу заздалегідь невідомий.** Повертає менеджер, якому реально належить цей квест: особистий — для Personal/Individual, party — для Shared. |

⚠️ Узяти `GetQuestManager()` для **Shared**-квесту — найчастіша помилка під час роботи з плагіном: повернеться менеджер гравця, у якому Shared-прогресу немає ніколи, і квест мовчки прочитається як «недоступний / не активний / не завершений», без жодної помилки будь-де. Використовуйте `GetQuestManagerForQuest()` усюди, де квест може виявитися Shared, або перевіряйте обидва менеджери, якщо збираєте список.

## Керування квестами

| Функція | Призначення |
|---|---|
| `StartQuest(Player, Quest)` | Запустити Personal-квест. Попереджає й завершується невдачею, якщо передано Shared-квест. |
| `StartPartyQuest(WorldContextObject, Quest, Participants)` | Запустити Shared/Individual квест із явним ростером. |
| `StartPartyQuestForAllPlayers(WorldContextObject, Quest)` | Запустити Shared/Individual квест на всіх поточних підключених гравцях — див. [06](06-Multiplayer.md). |
| `AddPartyQuestParticipant` / `RemovePartyQuestParticipant` | Керування ростером party-квесту. |
| `CompleteQuest(Player, Quest)` | Вручну завершити квест, готовий до здачі. Маршрутизує за режимом розподілу прогресу автоматично. |
| `FailQuest(Player, Quest)` | Провалити квест. |
| `AbandonQuest(Player, Quest)` | Скасування з ініціативи гравця (потребує `bCanAbandon`). |

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
| `CanStartQuest(Player, Quest)` | Пререквізити виконані + квест ще не активний/завершений. |
| `IsTargetRelevant(Player, EventID, TargetID)` | `true`, якщо у гравця є активна ціль, яку просунув би `NotifyQuestEvent(EventID, TargetID)` (перевіряє свій + party менеджер). Керує маркерами world-цілей. |
| `GetActiveQuests(Player)` | Активні Personal + Individual квести гравця. |
| `GetActiveSharedQuests(WorldContextObject)` | Активні Shared-квести з `GameState`. |
| `GetCompletedQuests(Player)` | Завершені квести гравця. |
| `GetFailedQuests(Player)` | Провалені квести гравця (вичерпався таймер, провалені вручну тощо). |
| `GetQuestProgress(Player, Quest, OutObjectives, OutProgress)` | Паралельні масиви всіх цілей та їхнього поточного прогресу. |
| `GetQuestCompletionPercentage(Player, Quest)` | `0.0`–`1.0`, лише за обов'язковими цілями. |
| `AreAllRequiredObjectivesComplete(Player, Quest)` | Та сама перевірка, що всередині використовує `IsQuestReadyToTurnIn`. |
| `GetQuestTimeRemaining(Player, Quest)` | Секунд лишилося у таймерного квесту (`0`, якщо не таймерний/не активний). |
| `GetQuestTimeRemainingText(Player, Quest)` | Те саме у форматі `MM:SS`. |
| `GetObjectiveTimeRemaining(Player, Quest, Objective)` | Секунд лишилося у таймерної *цілі* — незалежно від таймера самого квесту (див. [03](03-Authoring-Quests.md#таймерні-цілі)). |
| `GetObjectiveTimeRemainingText(Player, Quest, Objective)` | Те саме у форматі `MM:SS`. |

## Утиліти прогресу (без стану)

Корисні для UI-коду, у якого на руках лише `FDemoQuestObjectiveProgress` або значення enum, без пошуку менеджера:

| Функція | Повертає |
|---|---|
| `GetObjectiveProgressText(Progress)` | Текст на кшталт `"3/5"`. |
| `IsObjectiveCompleted(Progress)` | `bool`. |
| `GetObjectiveProgressPercentage(Progress)` | `0.0`–`1.0`. |
| `GetQuestStateText(State)` | Готовий до локалізації текст для `EDemoQuestState`. |
| `GetObjectiveStateText(State)` | Готовий до локалізації текст для `EDemoQuestObjectiveState`. |

## Утиліти party-квестів

| Функція | Призначення |
|---|---|
| `GetPartyQuestParticipants(WorldContextObject, Quest)` | Ростер Shared або Individual квесту. |
| `GetPlayerContribution(WorldContextObject, Quest, Player)` | Частка гравця в прогресі Shared-квесту. |
| `IsPlayerParticipant(WorldContextObject, Quest, Player)` | Перевірка членства в ростері. |
| `HasPlayerCompletedIndividualQuest(Player, Quest)` | Перевірка особистого завершення Individual-квесту. |

## Куди далі

- Низькорівневий API `UDemoQuestManagerComponent`, який огортають ці функції (делегати, нутрощі party-квестів): [01](01-Core-Concepts.md), [04](04-Event-Driven-Progress.md), [06](06-Multiplayer.md)
- Консольні команди, побудовані на тих самих запитах: [09 — Валідація та чити](09-Validation-And-Cheats.md)

<!-- doc-footer:start -->
---
*Generated 2026-08-04 17:36 UTC from `Docs/Full/` - do not edit this page directly.*
<!-- doc-footer:end -->
