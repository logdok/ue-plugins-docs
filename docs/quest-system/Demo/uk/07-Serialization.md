# 07 — Серіалізація

*[🇬🇧 English](../en/07-Serialization.md) | 🇺🇦 Українська*

QuestSystemDemo не постачає систему збережень — натомість він дає вам самодостатню структуру-знімок, спроєктовану для вбудовування в той `USaveGame`, який уже використовується у вашій грі.

## Експорт

```cpp
FDemoQuestSystemSerializedState State = QuestManager->ExportState();
```

`FDemoQuestSystemSerializedState` (`Core/DemoQuestSerialization.h`) містить усе необхідне для відновлення стану цього менеджера:

| Поле | Вміст |
|---|---|
| `ActiveQuests` | Шлях, стан, таймер і прогрес за кожною ціллю для кожного активного квесту (`FDemoActiveQuestSaveData` / `FDemoObjectiveProgressSaveData`). |
| `CompletedQuestPaths` / `FailedQuestPaths` | Шляхи до завершених/провалених ассетів квестів. |
| `FinishedQuests` | Фінальні знімки прогресу цілей для завершених/провалених квестів (`FDemoActiveQuestSaveData`) — реальний кінцевий стан (наприклад `2/3`, які опційні цілі були виконані) для журналу/історії. Відновлюються в `FinishedQuestRecords` менеджера; читаються через `GetObjectiveProgress` / `GetFinishedQuestObjectives`. |
| `RewardedQuestPaths` | Шляхи до квестів, що вже видали нагороди — **критично важливо**: запобігає повторній видачі нагород після перезавантаження. |
| `NotifiedAvailableQuestPaths` | Запобігає спаму сповіщеннями «доступний новий квест!» після перезавантаження. |
| `PartyQuestStates` | Заповнюється лише під час експорту party-менеджера (GameState). Ростер і дані про внесок, за рядковим ID гравця. |

Ассети зберігаються як `FSoftObjectPath`, тож стан переживає сесії й нічого не форс-завантажує, доки ви його не імпортуєте.

## Імпорт

```cpp
QuestManager->ImportState(SavedState);
```

Викликайте це **після** повної ініціалізації власного `PlayerState`/`GameState` (тобто після того, як сабсистема без налаштування — або ваш ручний код — створила менеджер). Імпорт спочатку очищає весь поточний прогрес квестів, потім відновлює його зі збережених шляхів, синхронно завантажуючи кожен ассет. Ассет квесту чи цілі, який не вдалося завантажити, пропускається й логується як попередження, а не як фатальна помилка.

## Вбудовування у свій save game

```cpp
UCLASS()
class UMySaveGame : public USaveGame
{
    GENERATED_BODY()
public:
    UPROPERTY()
    FDemoQuestSystemSerializedState PersonalQuestState;

    UPROPERTY()
    FDemoQuestSystemSerializedState PartyQuestState; // актуально, лише якщо ви також зберігаєте спільний стан
};

// Збереження
UMySaveGame* SaveGame = Cast<UMySaveGame>(UGameplayStatics::CreateSaveGameObject(UMySaveGame::StaticClass()));
SaveGame->PersonalQuestState = MyQuestManager->ExportState();
UGameplayStatics::SaveGameToSlot(SaveGame, TEXT("Slot1"), 0);

// Завантаження
if (UMySaveGame* Loaded = Cast<UMySaveGame>(UGameplayStatics::LoadGameFromSlot(TEXT("Slot1"), 0)))
{
    MyQuestManager->ImportState(Loaded->PersonalQuestState);
}
```

## Відоме обмеження: ростер групи

`PartyQuestStates` зберігає учасників і внески за «сирим» рядком player ID, а не за живими вказівниками `APlayerState*` — їх ще не існує в момент завантаження файлу збереження. Перетворення збереженого player ID назад на `PlayerState` гравця, який реально перепідключився, — за своєю природою задача, специфічна для конкретної гри (залежить від вашої системи логіна/ідентифікації), тому `ImportState` відновлює *суми* внеску, але повторне приєднання учасників лишає на ваш код. Для більшості ігор це важливо лише для Shared/Individual квестів, які мають пережити повний перезапуск сервера з гравцями, що перепідключаються — стан Personal-квестів і party-стан у межах однієї сесії спеціальної обробки не потребують.

## Куди далі

- Повний довідник функцій для всього використаного вище: [08 — Довідник Blueprint-бібліотеки](08-Blueprint-Library-Reference.md)

<!-- doc-footer:start -->
---
*Generated 2026-08-04 19:47 UTC from `Docs/Full/` - do not edit this page directly.*
<!-- doc-footer:end -->
