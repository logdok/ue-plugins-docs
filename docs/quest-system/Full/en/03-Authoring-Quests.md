# 03 — Authoring Quests

*🇬🇧 English | [🇺🇦 Українська](../uk/03-Authoring-Quests.md)*

Quests and objectives are `UPrimaryDataAsset`s. Create them in the Content Browser: **Add → Miscellaneous → Data Asset**, then pick `QuestObjectiveData` or `QuestData` as the data asset class.

Build objectives first, then reference them from a quest.

## UQuestObjectiveData

| Property | Type | Meaning |
|----------|------|---------|
| `ObjectiveName` | `FText` | Short display name ("Defeat the Bandit Leader"). |
| `ObjectiveDescription` | `FText` | Longer description for quest log UI. |
| `ObjectiveType` | `EQuestObjectiveType` | What kind of action completes this objective — see below. |
| `TargetCount` | `int32` (≥ 1) | How many times the action must happen. |
| `bIsOptional` | `bool` | Optional objectives don't block quest completion. |
| `bIsSequential` | `bool` | If true, this objective only becomes active once earlier sequential objectives in the same quest are done. See [04](04-Event-Driven-Progress.md). |
| `ObjectiveIdentifier` | `FName` | The value gameplay events must match (e.g. `"Bandit"`, `"HealthPotion"`). |
| `ExpectedEventID` | `FName` | Only used when `ObjectiveType == Custom`. See below. |
| `TimeLimit` | `float` seconds (default `0`) | `0` = no limit. Independent per-step deadline, separate from the quest's own `TimeLimit`. See "Timed quests" below. |
| `TimerWarningThreshold` | `float` seconds (default `0`) | `0` = no warning. Seconds remaining at which `OnObjectiveTimerWarning` fires once. Only meaningful while `TimeLimit > 0`. |
| `CustomData` | `TMap<FName, FString>` | Free-form key/value metadata for your own logic. |
| `ObjectiveIcon` | `UTexture2D*` | Optional UI icon. |

### Objective types and their matching event

| `ObjectiveType` | Matches event ID |
|---|---|
| `KillTarget` | `"Kill"` |
| `CollectItem` | `"Collect"` |
| `ReachLocation` | `"Location"` |
| `InteractWith` | `"Interact"` |
| `Custom` | Whatever `ExpectedEventID` says (or any event, if `ExpectedEventID` is unset) |

The matching mechanics are covered in full in [04 — Event-Driven Progress](04-Event-Driven-Progress.md) — that chapter also explains an important subtlety: two *different* objectives (in the same quest or in different quests) that share the same `(ObjectiveType, ObjectiveIdentifier)` pair will **both** be credited by a single matching event. That is intentional shared-credit behavior, not a bug — read that chapter before you rely on identifiers being globally unique.

### Custom objectives

`Custom` objectives exist for gameplay that doesn't fit Kill/Collect/Location/Interact — crafting, reputation gain, PvP wins, building structures, and so on. `ExpectedEventID` narrows which event category the objective accepts, so a `Custom` objective identified `"HealthPotion"` with `ExpectedEventID = "Craft"` only reacts to a crafting event, not to a `Collect` event that happens to carry the same identifier:

```cpp
// Objective: ObjectiveType = Custom, ObjectiveIdentifier = "HealthPotion", ExpectedEventID = "Craft"
UQuestBlueprintLibrary::NotifyCustomQuestEvent(PlayerState, "Craft", "HealthPotion", 1); // matches
UQuestBlueprintLibrary::NotifyCollectEvent(PlayerState, "HealthPotion", 1);              // ignored
```

If `ExpectedEventID` is left empty, the objective accepts *any* event ID carrying a matching identifier — maximum flexibility, less safety.

## UQuestData

| Property | Type | Meaning |
|----------|------|---------|
| `QuestID` | `FName` | Stable identifier used by `FindQuestByID` and save data. Set this — see [09](09-Validation-And-Cheats.md). |
| `QuestName` | `FText` | Display name. |
| `QuestDescription` | `FText` | Full narrative text. |
| `QuestSummary` | `FText` | Short summary for a quest log list. |
| `QuestCategory` | `FName` | Free-form filter tag ("MainStory", "SideQuest", …). |
| `Objectives` | `TArray<UQuestObjectiveData*>` | The objectives that make up this quest. |
| `PrerequisiteQuests` | `TArray<UQuestData*>` | Quests that must be completed first. Validated for cycles automatically — see [09](09-Validation-And-Cheats.md). |
| `CustomData` | `TMap<FName, FString>` | Free-form metadata (quest-giver NPC ID, quest-line tag, …). |
| `Rewards` | `FQuestReward` | See below. |
| `QuestIcon` | `UTexture2D*` | UI icon. |
| `QuestGiverDisplayName` | `FText` | Informational only. |
| `bCanAbandon` | `bool` (default `true`) | Whether the player can manually cancel it. |
| `bAutoComplete` | `bool` (default `false`) | `true`: completes instantly once all required objectives are done. `false`: the player must actively turn it in (see [05 — World Components](05-World-Components.md), `UQuestReceiverComponent`). |
| `TimeLimit` | `float` seconds (default `0`) | `0` = no limit. See "Timed quests" below. |
| `TimerWarningThreshold` | `float` seconds (default `0`) | `0` = no warning. Seconds remaining at which `OnQuestTimerWarning` fires once. Only meaningful while `TimeLimit > 0`. |
| `QuestSharingMode` | `EQuestSharingMode` | Personal / Shared / Individual — see [01](01-Core-Concepts.md). |
| `bAutoEnrollNewParticipants` | `bool` (default `true`) | Shared/Individual only — see [06](06-Multiplayer.md). |
| `MinPartySize` / `MaxPartySize` | `int32` | Party size limits (Shared/Individual). `0` means "no limit" for `MaxPartySize`; `MinPartySize` of `0` or `1` allows solo starts. |

### FQuestReward

```cpp
USTRUCT(BlueprintType)
struct FQuestReward
{
    TArray<TSubclassOf<AActor>> ItemRewards;
    TMap<FName, int32> RewardAmounts; // e.g. "Gold" -> 50, "Carrots" -> 12, "ExperiencePoints" -> 100
};
```

Deliberately has no built-in notion of currency, XP, or any other named resource — what a "reward" even *is* (gold? carrots? reputation? skill points?) is entirely project-defined. Two generic channels cover every shape of reward: `RewardAmounts` for anything expressed as a keyed number, `ItemRewards` for anything that's a discrete spawned/given actor.

`FQuestReward` is only *data* — nothing spends it automatically, and the plugin never reads or interprets a `RewardAmounts` key itself. Implement the actual granting in `GiveQuestRewards()`, routing each key to whatever it means in your game; see [02 — Zero-Config Integration](02-Zero-Config-Integration.md).

### Auto-complete vs. turn-in

- `bAutoComplete = true`: the quest completes (and rewards fire) the instant the last required objective finishes. Good for simple collection/kill quests.
- `bAutoComplete = false`: the quest becomes *ready to turn in* (`IsQuestReadyToTurnIn()` returns `true`) but stays `Active` until something calls `CompleteQuest()` — typically a `UQuestReceiverComponent` on an NPC or chest. Good for quests with a narrative payoff at a specific location.

### Timed quests

Set `TimeLimit` to a positive number of seconds. Once started, `UQuestManagerComponent` counts down `TimeRemaining` on that quest every tick. Two things happen automatically:

- `OnQuestTimerWarning(Quest, TimeRemaining)` fires once when the remaining time drops to `TimerWarningThreshold` (`0` = no warning, set per quest — there's no longer a project-wide default) — use it to flash a UI warning.
- If the timer reaches zero before the quest completes, the quest **fails automatically** (`FailQuest` is called for you).

The demo map's `DQ_TimedDelivery` quest (`Content/Demo/DataAssets/Quests/`) is a working example: `TimeLimit = 45`, `bAutoComplete = true`, a single `ReachLocation` objective.

### Timed objectives

Objectives can carry their own `TimeLimit`/`TimerWarningThreshold`, completely independent of the quest's own — a quest can be untimed overall and still have one 30-second "hold the line" step, or vice versa. The countdown only runs while that specific objective is `Active`, and starts fresh each time an objective activates (including a later sequential one).

What happens when an objective's own timer reaches zero depends on `bIsOptional`:

- **Required** objective: the whole quest fails (`FailQuest`), exactly like the quest-level `TimeLimit` expiring.
- **Optional** objective: only that objective is marked `Failed` — the quest keeps going, and can still complete once its required objectives are done (the same `bIsOptional` rule that already excludes it from the completion check).

`OnObjectiveTimerWarning(Quest, Objective, TimeRemaining)` mirrors `OnQuestTimerWarning` at the objective level — see [04 — Event-Driven Progress](04-Event-Driven-Progress.md#listening-for-progress).

Both the quest-level and every timed objective's remaining time are visible live in the `QuestSystemDebug` overlay's Quests tab — see [09 — Validation & Cheats](09-Validation-And-Cheats.md).

### Prerequisites and validation

Any quest can list other quests in `PrerequisiteQuests`; `CanStartQuest()` refuses to start it until every prerequisite is completed. The dependency graph is validated automatically in the editor (on edit and on save) to catch cycles — see [09 — Validation & Cheats](09-Validation-And-Cheats.md) for the full validation and debugging toolset.

Prerequisites can freely cross sharing modes (e.g. a Personal quest requiring a Shared one) — completion is checked on whichever manager actually owns *that specific prerequisite*, not blindly on the checking quest's own manager, so a Shared prerequisite's completion (tracked on the party/GameState manager) is still recognized by a Personal quest's check. The one direction this doesn't resolve for you: a **Shared** quest requiring a **Personal/Individual** prerequisite — "whose personal completion should count for the whole party" is a design call this plugin leaves to you, so such a prerequisite is checked against the party manager's own completions (which a per-player quest never joins) and will never be satisfied automatically.

## The demo's quests

`Content/Demo/DataAssets/{Quests,Objectives}/` ships a working, playable set of quests exercising everything above — a concrete reference alongside the property tables. `QuestID`/`ObjectiveIdentifier` are the stable, functional keys (matched by `FindQuestByID` and by gameplay events respectively); asset names and display `Name`s are cosmetic and can be renamed freely without touching them.

### Quests

| Asset | `QuestID` | Name | Category | Sharing mode | Objectives |
|---|---|---|---|---|---|
| `DQ_AlchemistApprentice` | `Quest_AlchemistApprentice` | Alchemist's Apprentice | Crafting | Personal | `DO_CraftHealthPotions`, `DO_CraftFungi`, `DO_CraftManaPotion` |
| `DQ_BanditThreat` | `Quest_BanditThreat` | Bandit Threat | MainStory | Personal | `DO_KillBandits` |
| `DQ_CrystalWarmup` | `Quest_CrystalWarmup` | Crystal Warmup | DemoV2 | Personal — prereq `DQ_TimedDelivery` | `DO_CollectCrystals` |
| `DQ_HerbGathering` | `Quest_HerbGathering` | Healing Herbs | — | Personal | `DO_CollectHerbs` |
| `DQ_IntroQuest` | `Quest_Intro` | A Troubled Village | MainStory | Personal | `DO_TalkToElder` |
| `DQ_PartyHarvest` | `Quest_PartyHarvest` | Party Harvest | DemoV2 | Shared | `DO_HarvestBerries` |
| `DQ_Pilgrimage` | `Quest_Pilgrimage` | Pilgrimage | DemoV2 | Individual | `DO_ReachShrine` |
| `DQ_TargetPractice` | `Quest_TargetPractice` | Target Practice | DemoV2 | Personal — prereq `DQ_CrystalWarmup`, manual turn-in | `DO_KillDummies` |
| `DQ_TimedDelivery` | `Quest_TimedDelivery` | Timed Delivery | DemoV2 | Personal — `TimeLimit = 45s` | `DO_ReachDelivery` |
| `DQ_TreasureHunt` | `Quest_TreasureHunt` | The Treasure Hunt | — | Personal — sequential, manual turn-in | `DO_FindMap` → `DO_DecodeMap` → `DO_LocateTreasure` |
| `DQ_UrgentDelivery` | `Quest_UrgentDelivery` | Urgent Delivery | — | Personal — `TimeLimit = 25s` | `DO_DeliverMedicine`, `DO_LocateChild` |

### Objectives

| Asset | `ObjectiveIdentifier` | Name | Type | Target |
|---|---|---|---|---|
| `DO_CollectCrystals` | `EnergyCrystal` | Collect Energy Crystals | CollectItem | 3 |
| `DO_CollectHerbs` | `MagicalHerb` | Collect Magical Herbs | CollectItem | 4 |
| `DO_CraftFungi` | `Fungi` | Craft Fungi | Custom (`ExpectedEventID="Craft"`) | 2 |
| `DO_CraftHealthPotions` | `HealthPotion` | Craft Health Potions | Custom (`ExpectedEventID="Craft"`) | 3 |
| `DO_CraftManaPotion` | `ManaPotion` | Craft Mana Potion | Custom (`ExpectedEventID="Craft"`) | 1 |
| `DO_DecodeMap` | `Decoder` | Decode the Map | InteractWith | 1 |
| `DO_DeliverMedicine` | `SickChild` | Deliver Medicine | CollectItem | 2 |
| `DO_FindMap` | `AncientMap` | Find the Ancient Map | CollectItem | 1 |
| `DO_HarvestBerries` | `PartyBerry` | Harvest Party Berries | CollectItem | 6 |
| `DO_KillBandits` | `Bandit` | Defeat Bandits | KillTarget | 5 |
| `DO_KillDummies` | `TrainingDummy` | Destroy Training Dummies | KillTarget | 3 |
| `DO_LocateChild` | `ChildLocation` | Find the Sick Child | ReachLocation | 1 |
| `DO_LocateTreasure` | `TreasureLocation` | Locate the Treasure | ReachLocation | 1 |
| `DO_ReachDelivery` | `DeliveryPoint` | Reach the Delivery Point | ReachLocation | 1 |
| `DO_ReachShrine` | `AncientShrine` | Reach the Ancient Shrine | ReachLocation | 1 |
| `DO_TalkToElder` | `VillageElder` | Speak with the Elder | InteractWith | 1 |

## Where to go next

- Wiring gameplay events to these objectives: [04 — Event-Driven Progress](04-Event-Driven-Progress.md)
- Placing quest-giving actors in your level: [05 — World Components](05-World-Components.md)

<!-- doc-footer:start -->
---
*Last updated: 2026-08-04 15:08 UTC*
<!-- doc-footer:end -->
