# 06 — Мультиплеєр

*[🇬🇧 English](../en/06-Multiplayer.md) | 🇺🇦 Українська*

Підтримка мультиплеєра — не надбудова згори, а причина, з якої плагін влаштований саме так (див. [01 — Основні поняття](01-Core-Concepts.md) про модель «один компонент — два місця»). Цей розділ описує API party-квестів і патерн, який дозволяє *одній і тій самій* карті та *одному й тому самому* контенту працювати і в одиночній грі, і в мультиплеєрі.

## Авторитет сервера

Кожен метод `UDemoQuestManagerComponent`, що змінює стан (`StartQuest`, `CompleteQuest`, `FailQuest`, `AbandonQuest`, `UpdateObjectiveProgress`, `SetObjectiveProgress`, `NotifyQuestEvent`, `AddParticipant`, `RemoveParticipant`, …), усередині себе перевіряє авторитет і при виклику з клієнта перекидається на сервер. Думати про це не потрібно ніколи — викликайте публічний метод звідки завгодно, з клієнта чи сервера, і все спрацює правильно, **зокрема й при виклику на party-менеджері для Shared/Individual квесту**: у `GameState` немає єдиного власного клієнтського з'єднання, тож reliable RPC не можна оголосити прямо на компоненті, розміщеному на `GameState` (рушій мовчки відмовляється виконувати його для будь-якого клієнта) — такі виклики натомість перенаправляються через власний менеджер локального гравця. Це суто деталь реалізації, налаштовувати нічого не потрібно.

Варто знати одне застереження: на клієнті ці методи повертають `true` оптимістично, щойно запит надіслано, а не тоді, коли сервер справді його прийняв. Якщо виклик потенційно може бути відхилений на сервері, орієнтуйтеся на делегати (`OnQuestStateChanged`, `OnQuestCompleted`, …) чи подальшу перевірку стану як на справжнє джерело істини — не будуйте UI виключно на значенні, що повертається.

`ActiveQuests` реплікується автоматично (`OnRep_ActiveQuests`); `CompletedQuests`/`FailedQuests`/`PartyQuestStates` — теж. Усі вони реплікуються кожному спостерігачеві актора-хоста — тобто на менеджері `PlayerState` дані квестів гравця доходять до всіх клієнтів, а не лише до власника. Це тому, що той самий клас компонента зберігає колективний Shared-прогрес на `GameState`, у якого немає власника, — а отже, суцільний `COND_OwnerOnly` неможливий. Вважайте записи інших гравців побічними/read-only; якщо потрібна сувора прив'язка до власника, розділіть компонент на два підкласи.

## Запуск party-квесту

Shared та Individual квести запускаються зі **списком учасників**, а не з одним гравцем:

```cpp
// Явний список учасників
UDemoQuestManagerComponent* PartyManager = UDemoQuestBlueprintLibrary::GetPartyQuestManager(this);
PartyManager->StartPartyQuest(QuestData, { Player1State, Player2State });

// Або через Blueprint-бібліотеку (працює звідки завгодно з world context):
UDemoQuestBlueprintLibrary::StartPartyQuest(this, QuestData, Participants);
```

### Патерн «одиночна гра + мультиплеєр»

`StartPartyQuestForAllPlayers()` запускає party-квест на **всіх, хто зараз підключений**:

```cpp
UDemoQuestBlueprintLibrary::StartPartyQuestForAllPlayers(this, SharedQuestData);
```

У standalone-режимі це група з одного — квест поводиться точно як Personal-квест для цього єдиного гравця. На сервері з кількома підключеними гравцями той самий виклик зараховує їх усіх. Саме так поводиться `UDemoQuestGiverComponent::GiveQuest()` для Shared/Individual квестів, і саме тому `UDemoQuestInteractorComponent`, що взаємодіє з giver'ом, «просто працює» в обох режимах — дизайнерові рівня нічого міняти не потрібно.

Скомбінуйте це з `bAutoEnrollNewParticipants` (на ассеті квесту, див. [03](03-Authoring-Quests.md)), щоб підхоплювати й гравців, які підключилися **після** старту квесту — `UDemoQuestWorldSubsystem` автоматично викликає `EnrollPlayerInAutoJoinQuests()` для кожного нового `PlayerState`, тож drop-in кооп працює без жодного рядка додаткового коду.

`MinPartySize` / `MaxPartySize` на ассеті квесту примусово перевіряються `StartPartyQuest` — використовуйте `MinPartySize`, щоб змусити квест вимагати справжнього кооперативу, навіть якщо API з радістю запустив би його й соло.

## Керування учасниками

```cpp
PartyManager->AddParticipant(QuestData, NewPlayerState);
PartyManager->RemoveParticipant(QuestData, LeavingPlayerState);   // провалює Shared-квест, якщо учасників не лишилося

TArray<APlayerState*> Participants = PartyManager->GetParticipants(QuestData);
bool bIn = PartyManager->IsParticipant(QuestData, SomePlayer);
```

## Shared-квести: колективний прогрес + внесок

У Shared-квесту рівно один набір лічильників цілей, реплікований з `GameState`. Події кожного учасника додаються до *тих самих* лічильників (з урахуванням правил зіставлення за ідентифікатором із [04](04-Event-Driven-Progress.md)), а індивідуальна частка кожного гравця при цьому відстежується окремо — для таблиць лідерів або бонусних нагород:

```cpp
int32 MyKills = PartyManager->GetPlayerContribution(QuestData, MyPlayerState);
```

## Individual-квести: роздільний прогрес, спільна межа завершення

Кожен учасник отримує свою власну копію цілей на своєму `PlayerState` — `AddParticipant` для Individual-квесту також викликає `StartQuest` для цього гравця. `GameState` відстежує лише те, хто вже закінчив:

```cpp
bool bDone = UDemoQuestBlueprintLibrary::HasPlayerCompletedIndividualQuest(MyPlayerState, QuestData);
```

Усередині, коли гравець завершує свою копію, `MarkParticipantCompleted()` фіксує це на party-менеджері; щойно позначені всі учасники, службові дані квесту на боці групи автоматично очищаються. Якщо гравець відключився, а party-менеджера немає (рідкісний випадок суто локальних налаштувань), квест усе одно завершиться локально для цього гравця — див. [04](04-Event-Driven-Progress.md), як маршрутизація коректно відкочується в цьому випадку.

## Куди далі

- Збереження прогресу квестів між сесіями (зокрема те, що *не* зберігається — ростер групи): [07 — Серіалізація](07-Serialization.md)

<!-- doc-footer:start -->
---
*Generated 2026-08-05 14:13 UTC from `Docs/Full/` - do not edit this page directly.*
<!-- doc-footer:end -->
