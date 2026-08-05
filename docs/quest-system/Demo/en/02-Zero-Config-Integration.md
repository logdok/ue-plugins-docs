# 02 — Zero-Config Integration

*🇬🇧 English | [🇺🇦 Українська](../uk/02-Zero-Config-Integration.md)*

You do not need a custom `PlayerState`, `GameState`, or `GameMode` class to use QuestSystemDemo. This chapter explains what does the wiring for you, and how to customize it when you need to.

## UDemoQuestWorldSubsystem

`UDemoQuestWorldSubsystem` is a `UWorldSubsystem` — it exists automatically in every game world, no setup required. On the server (a standalone single-player session counts as a server), it:

- creates a `UDemoQuestManagerComponent` on the **GameState** (the party manager);
- creates a `UDemoQuestManagerComponent` on **every `PlayerState`** as it spawns — this covers login, late join, seamless travel, and PIE;
- enrolls newly-joined players into active party quests that have `bAutoEnrollNewParticipants` set (see [06 — Multiplayer](06-Multiplayer.md));
- optionally validates the whole quest dependency graph on startup and logs the result (development builds only).

If a `PlayerState` or `GameState` **already has** a `UDemoQuestManagerComponent` — because you placed one manually in a custom class or Blueprint — the subsystem leaves it alone. Manual and zero-config setups can coexist.

```cpp
// Get the subsystem directly if you need it:
UDemoQuestWorldSubsystem* Subsystem = UDemoQuestWorldSubsystem::Get(this);
UDemoQuestManagerComponent* PartyManager = Subsystem->GetPartyQuestManager();
UDemoQuestManagerComponent* MyManager = UDemoQuestWorldSubsystem::GetPlayerQuestManager(MyPlayerState);
```

In most code you will use `UDemoQuestBlueprintLibrary` instead (see [08](08-Blueprint-Library-Reference.md)) — it wraps the same lookups with clearer names (`GetQuestManager`, `GetPartyQuestManager`).

## UDemoQuestSystemSettings

Project Settings → **Game → Quest System (Demo)** (stored in `DefaultGame.ini` under `[/Script/QuestSystemDemo.DemoQuestSystemSettings]`):

| Property | Default | Meaning |
|----------|---------|---------|
| `bAutoCreatePlayerQuestManagers` | `true` | Create a manager on every `PlayerState`. |
| `bAutoCreatePartyQuestManager` | `true` | Create a manager on the `GameState`. Required for Shared/Individual quests. |
| `QuestManagerClass` | `UDemoQuestManagerComponent` | The class the subsystem instantiates. Point this at your own subclass — see below. |
| `bValidateQuestsOnStartup` | `true` | Run dependency validation when a world starts (non-shipping builds only). |

Timer warning thresholds are authored per quest/objective asset (`TimerWarningThreshold`, `0` = no warning) rather than as a single project-wide value — see [03 — Authoring Quests](03-Authoring-Quests.md#timed-quests).

```ini
; Example: DefaultGame.ini
[/Script/QuestSystemDemo.DemoQuestSystemSettings]
bAutoCreatePartyQuestManager=false
```

## Overriding reward logic

`GiveQuestRewards()` is a `BlueprintNativeEvent` on `UDemoQuestManagerComponent` — the natural place to grant XP, gold, or items. Because the subsystem creates the manager for you, you don't subclass `PlayerState` to reach it; instead, subclass the **component** and point `QuestManagerClass` at your subclass. Either a C++ subclass or a Blueprint one works — pick whichever fits your team.

### C++ version

```cpp
// MyQuestManagerComponent.h
UCLASS()
class UMyQuestManagerComponent : public UDemoQuestManagerComponent
{
    GENERATED_BODY()
protected:
    virtual void GiveQuestRewards_Implementation(UDemoQuestData* QuestData) override;
};

// MyQuestManagerComponent.cpp
void UMyQuestManagerComponent::GiveQuestRewards_Implementation(UDemoQuestData* QuestData)
{
    if (APlayerState* PS = GetOwningPlayerState())
    {
        if (AMyPlayerState* MyPS = Cast<AMyPlayerState>(PS))
        {
            // RewardAmounts is a plain TMap<FName, int32> - the plugin has no
            // built-in notion of "gold" or "XP", so route whichever keys your
            // own quest data uses to whatever they mean in your game.
            if (const int32* XP = QuestData->Rewards.RewardAmounts.Find(TEXT("ExperiencePoints")))
            {
                MyPS->AddExperience(*XP);
            }
            if (const int32* Gold = QuestData->Rewards.RewardAmounts.Find(TEXT("Gold")))
            {
                MyPS->AddGold(*Gold);
            }
        }
    }
    // Called on both the PlayerState manager (Personal/Individual) and the
    // GameState manager (Shared) — GetOwningPlayerState() returns nullptr on
    // the party manager, so branch on IsPartyQuestManager() if you need to
    // reward every participant of a Shared quest.
}
```

Then set **QuestManagerClass = MyQuestManagerComponent** in Project Settings. No other class needs to change — this works whether the component ends up on a vanilla `PlayerState` or a custom one.

### Blueprint version (no C++ required)

`GiveQuestRewards` being a `BlueprintNativeEvent` means a Blueprint class can override it exactly like a C++ subclass can — no engine-side special case needed:

1. Content Browser → **Add → Blueprint Class**. In the parent-class picker, enable **All Classes** and search for **Quest Manager Component** — pick it as the parent.
2. Name it, e.g. `BP_MyQuestManagerComponent`, and open it.
3. In the Event Graph, right-click → **Add Event** → search **Give Quest Rewards**. This places an `Event Give Quest Rewards` node with a `Quest Data` output pin.
4. Wire your reward logic from there: `Break` the `Quest Data` pin's `Rewards` struct to get `Item Rewards` / `Reward Amounts`, `Find` a key in `Reward Amounts` (e.g. `"ExperiencePoints"`, `"Gold"`, or whatever your project calls its resources) to pull out its amount, then call whatever functions your player-stats or inventory system exposes (e.g. `Get Player State` → cast to your custom `PlayerState` → `Add Experience`).
5. Project Settings → **Game → Quest System (Demo)** → set `Quest Manager Class` to `BP_MyQuestManagerComponent`.

The default C++ implementation (logging only) does not run unless you explicitly add **Call to Parent Function** on the event node — skipping it is fine, you lose nothing but the log line.

## Manual placement (opt-out)

If you'd rather keep full control, disable the relevant `bAutoCreate*` setting and create the component yourself, exactly as before the zero-config subsystem existed:

```cpp
AMyPlayerState::AMyPlayerState()
{
    QuestManagerComponent = CreateDefaultSubobject<UDemoQuestManagerComponent>(TEXT("QuestManagerComponent"));
    QuestManagerComponent->SetIsReplicated(true);
}
```

Both approaches produce an identical, fully-functional component — zero-config is simply the recommended default.

<!-- doc-footer:start -->
---
*Generated 2026-08-05 11:23 UTC from `Docs/Full/` - do not edit this page directly.*
<!-- doc-footer:end -->
