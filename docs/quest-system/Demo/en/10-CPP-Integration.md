# 10 — C++ Integration

*🇬🇧 English | [🇺🇦 Українська](../uk/10-CPP-Integration.md)*

Everything so far works from Blueprint alone. This chapter is for programmers extending the plugin from C++.

## Module dependencies

The plugin ships four modules, all under `Plugins/QuestSystemDemo/Source/`; add whichever your own module's `.Build.cs` actually uses to its `PublicDependencyModuleNames`/`PrivateDependencyModuleNames`:

| Module | What's in it |
|---|---|
| `QuestSystemDemo` | Core: `UDemoQuestData`/`UDemoQuestObjectiveData`, `UDemoQuestManagerComponent`, `UDemoQuestWorldSubsystem`, `UDemoQuestBlueprintLibrary`, settings, validation. Include `DemoQuestSystemTypes.h` for all of it. |
| `QuestSystemDemoWorld` | World-placed actors/components: `UDemoQuestGiverComponent`/`UDemoQuestReceiverComponent` (this chapter's first example subclasses one), the quest markers, `UDemoQuestInteractorComponent`, `UDemoQuestTargetComponent`, and the ready-made actors (see [05](05-World-Components.md)). Depends on `QuestSystemDemo`; also depends on `UMG` (for the widget marker), so pull it in only where you actually need these classes. Include `DemoQuestSystemWorldTypes.h` for all of it. |
| `QuestSystemDemoDebug` | Optional dev tooling: the debug overlay and cheat commands. Depends on `QuestSystemDemo` + `UMG`/`Slate`. Include its headers directly (no combined master include). |
| `QuestSystemTests` | Editor-only automation tests — not something a consuming project depends on. |

The demo host's own `HostQuestSystem.Build.cs` is a working example: `QuestSystemDemo` + `QuestSystemDemoWorld` + `QuestSystemDemoDebug` in `PrivateDependencyModuleNames`.

## Custom reward logic

Covered in full in [02 — Zero-Config Integration](02-Zero-Config-Integration.md): subclass `UDemoQuestManagerComponent`, override `GiveQuestRewards_Implementation`, and point `QuestManagerClass` at your subclass in Project Settings. That single override is called exactly once per quest (duplicate-protected), on whichever manager completed the quest.

```cpp
void UMyQuestManagerComponent::GiveQuestRewards_Implementation(UDemoQuestData* QuestData)
{
    const FDemoQuestReward& Rewards = QuestData->Rewards;

    if (IsPersonalQuestManager())
    {
        if (AMyPlayerState* MyPS = Cast<AMyPlayerState>(GetOwningPlayerState()))
        {
            // RewardAmounts is a plain TMap<FName, int32> - route whichever
            // keys your own quest data uses to whatever they mean in your game.
            if (const int32* XP = Rewards.RewardAmounts.Find(TEXT("ExperiencePoints")))
            {
                MyPS->AddExperience(*XP);
            }
            if (const int32* Gold = Rewards.RewardAmounts.Find(TEXT("Gold")))
            {
                MyPS->AddGold(*Gold);
            }
            for (const TSubclassOf<AActor>& ItemClass : Rewards.ItemRewards)
            {
                MyPS->GiveItem(ItemClass);
            }
        }
    }
    else if (IsPartyQuestManager())
    {
        // A Shared quest completed on the GameState - reward every participant.
        if (const FDemoPartyQuestState* State = FindPartyQuestState(QuestData))
        {
            for (APlayerState* Participant : State->Participants)
            {
                // ... reward each participant
            }
        }
    }
}
```

Do not put reward-granting logic in an `OnQuestCompleted` handler — that delegate fires *after* rewards and has no duplicate protection; it exists for UI/FX/analytics only.

## Extending Giver / Receiver components

Both `UDemoQuestGiverComponent` and `UDemoQuestReceiverComponent` expose their customization points as `BlueprintNativeEvent`s, so a C++ subclass overrides the `_Implementation` version exactly like any other native event:

```cpp
UCLASS()
class UMyQuestGiverComponent : public UDemoQuestGiverComponent
{
    GENERATED_BODY()
protected:
    virtual bool ShouldOfferQuestToPlayer_Implementation(UDemoQuestData* Quest, APlayerController* Player) const override
    {
        // e.g. gate by player level stored on a custom PlayerState. There's no
        // built-in "recommended level" field - use a CustomData convention instead.
        if (const AMyPlayerState* MyPS = Player ? Player->GetPlayerState<AMyPlayerState>() : nullptr)
        {
            const FString* RequiredLevelStr = Quest->CustomData.Find(TEXT("RequiredLevel"));
            const int32 RequiredLevel = RequiredLevelStr ? FCString::Atoi(**RequiredLevelStr) : 1;
            return MyPS->GetLevel() >= RequiredLevel;
        }
        return Super::ShouldOfferQuestToPlayer_Implementation(Quest, Player);
    }
};
```

## Sending events from your own systems

If your game already has combat, inventory, or dialogue systems, call the notify functions directly from there instead of using the ready-made world actors from [05](05-World-Components.md):

```cpp
// In your damage/death handling, once an enemy actually dies:
void AMyEnemy::Die(APlayerController* Killer)
{
    if (Killer && Killer->PlayerState)
    {
        UDemoQuestBlueprintLibrary::NotifyKillEvent(Killer->PlayerState, EnemyTypeTag);
    }
}

// In your inventory system, once an item is actually added:
void UMyInventoryComponent::AddItem(FName ItemID, int32 Count)
{
    // ... your inventory logic ...
    if (APlayerState* PS = GetOwningPlayerState())
    {
        UDemoQuestBlueprintLibrary::NotifyCollectEvent(PS, ItemID, Count);
    }
}
```

Fire the event only once the underlying game state actually changed (the enemy is really dead, the item is really in the inventory) — the quest system trusts the event as truth and has no way to verify it against your gameplay state.

## Reading progress from C++ (custom UI)

```cpp
UDemoQuestManagerComponent* Manager = UDemoQuestBlueprintLibrary::GetQuestManager(PlayerState);
for (const FDemoActiveQuest& Quest : Manager->GetActiveQuests())
{
    if (Quest.State != EDemoQuestState::Active) { continue; }

    for (UDemoQuestObjectiveData* Objective : Quest.QuestData->Objectives)
    {
        const FDemoQuestObjectiveProgress Progress = Manager->GetObjectiveProgress(Quest.QuestData, Objective);
        // Progress.CurrentProgress / Progress.TargetProgress / Progress.State
    }
}
```

The `QuestSystemDemoDebug` module's overlay (`DemoQuestDebugQuestCard.cpp`/`DemoQuestDebugObjectiveCard.cpp` in `Plugins/QuestSystemDemo/Source/QuestSystemDemoDebug/Private/`) is a complete, working example of exactly this pattern at scale — reading every quest's objectives, progress, and state to build C++-only UMG widgets with no WBP assets at all — worth reading if you're building your own quest UI in C++.

## Delegates, not polling

Whenever you have a specific instance to react to, prefer binding to the manager's delegates (listed in [04 — Event-Driven Progress](04-Event-Driven-Progress.md)) over polling `GetActiveQuests()` every frame:

```cpp
QuestManager->OnObjectiveUpdated.AddDynamic(this, &AMyHUD::HandleObjectiveUpdated);
QuestManager->OnQuestCompleted.AddDynamic(this, &AMyHUD::HandleQuestCompleted);
```

## See also

- [02 — Zero-Config Integration](02-Zero-Config-Integration.md) for how your custom classes get plugged in without touching `PlayerState`/`GameState`.
- [06 — Multiplayer](06-Multiplayer.md) for the party-quest API used when rewarding Shared/Individual quests.

<!-- doc-footer:start -->
---
*Generated 2026-08-04 19:24 UTC from `Docs/Full/` - do not edit this page directly.*
<!-- doc-footer:end -->
