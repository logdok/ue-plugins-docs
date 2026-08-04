# 07 — Serialization

*🇬🇧 English | [🇺🇦 Українська](../uk/07-Serialization.md)*

QuestSystemDemo does not ship a save system — it gives you a self-contained snapshot struct designed to be embedded into whatever `USaveGame` your game already uses.

## Exporting

```cpp
FDemoQuestSystemSerializedState State = QuestManager->ExportState();
```

`FDemoQuestSystemSerializedState` (`Core/DemoQuestSerialization.h`) contains everything needed to restore this manager's state:

| Field | Contents |
|---|---|
| `ActiveQuests` | Each active quest's path, state, timer, and per-objective progress (`FDemoActiveQuestSaveData` / `FDemoObjectiveProgressSaveData`). |
| `CompletedQuestPaths` / `FailedQuestPaths` | Paths to completed/failed quest assets. |
| `FinishedQuests` | Final per-objective snapshots for completed/failed quests (`FDemoActiveQuestSaveData`) — the real end-state (e.g. `2/3`, which optional objectives were done) for a journal/history UI. Restored into the manager's `FinishedQuestRecords`; read it back with `GetObjectiveProgress` / `GetFinishedQuestObjectives`. |
| `RewardedQuestPaths` | Paths to quests that already gave rewards — **critical**: prevents duplicate rewards after a reload. |
| `NotifiedAvailableQuestPaths` | Prevents "new quest available!" notification spam after a reload. |
| `PartyQuestStates` | Only populated when exporting the party manager (GameState). Roster + contribution data, by player ID string. |

Assets are stored as `FSoftObjectPath`, so the state survives across sessions and doesn't force-load anything until you import it.

## Importing

```cpp
QuestManager->ImportState(SavedState);
```

Call this **after** the owning `PlayerState`/`GameState` is fully initialized (i.e. after the zero-config subsystem — or your manual code — has created the manager). Importing clears all current quest progress first, then rebuilds it from the saved paths, loading each asset synchronously. A quest or objective asset that fails to load is skipped and logged as a warning, not treated as fatal.

## Embedding in your own save game

```cpp
UCLASS()
class UMySaveGame : public USaveGame
{
    GENERATED_BODY()
public:
    UPROPERTY()
    FDemoQuestSystemSerializedState PersonalQuestState;

    UPROPERTY()
    FDemoQuestSystemSerializedState PartyQuestState; // only relevant if you also persist shared state
};

// Saving
UMySaveGame* SaveGame = Cast<UMySaveGame>(UGameplayStatics::CreateSaveGameObject(UMySaveGame::StaticClass()));
SaveGame->PersonalQuestState = MyQuestManager->ExportState();
UGameplayStatics::SaveGameToSlot(SaveGame, TEXT("Slot1"), 0);

// Loading
if (UMySaveGame* Loaded = Cast<UMySaveGame>(UGameplayStatics::LoadGameFromSlot(TEXT("Slot1"), 0)))
{
    MyQuestManager->ImportState(Loaded->PersonalQuestState);
}
```

## Known limitation: party rosters

`PartyQuestStates` stores participants and contributions by a raw player-ID string, not by live `APlayerState*` pointers — those don't exist yet when you're loading a save file. Converting a saved player ID back into an actual reconnecting player's `PlayerState` is inherently game-specific (it depends on your login/identity system), so `ImportState` restores contribution *amounts* but leaves reattaching participants to your own code. For most games this only matters for Shared/Individual quests that must survive a full server restart with reconnecting players — Personal-quest state and single-session party state need no special handling.

## Where to go next

- The full function reference for everything used above: [08 — Blueprint Library Reference](08-Blueprint-Library-Reference.md)
