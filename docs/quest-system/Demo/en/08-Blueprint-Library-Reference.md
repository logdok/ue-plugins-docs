# 08 — Blueprint Library Reference

*🇬🇧 English | [🇺🇦 Українська](../uk/08-Blueprint-Library-Reference.md)*

`UDemoQuestBlueprintLibrary` is the recommended entry point for almost everything: every function is `static`, callable from Blueprint and C++ alike, and takes an `APlayerState*` (or a world context) so it can route to the correct manager (`PlayerState` vs `GameState`) for you — you rarely need to look up a `UDemoQuestManagerComponent` yourself.

All functions live in `Core/DemoQuestBlueprintLibrary.h`.

## Manager access

| Function | Returns |
|---|---|
| `GetQuestManager(Player)` | The player's personal manager (Personal + Individual quests). |
| `GetQuestManagerForController(PlayerController)` | Same, resolved from a controller. |
| `GetPartyQuestManager(WorldContextObject)` | The `GameState` manager (Shared quests). |
| `GetQuestManagerForQuest(Player, Quest)` | **The one to reach for when the sharing mode isn't already known.** Returns whichever manager actually owns that quest: the player's own for Personal/Individual, the party one for Shared. |

⚠️ Picking `GetQuestManager()` for a **Shared** quest is the single most common mistake with this plugin: it returns the player's manager, which never holds Shared progress, so the quest silently reads as "not available / not active / not completed" with no error anywhere. Use `GetQuestManagerForQuest()` whenever the quest could be Shared, or check both managers when aggregating a list.

## Quest management

| Function | Purpose |
|---|---|
| `StartQuest(Player, Quest)` | Start a Personal quest. Warns and fails if given a Shared quest. |
| `StartPartyQuest(WorldContextObject, Quest, Participants)` | Start a Shared/Individual quest with an explicit roster. |
| `StartPartyQuestForAllPlayers(WorldContextObject, Quest)` | Start a Shared/Individual quest with everyone currently connected — see [06](06-Multiplayer.md). |
| `AddPartyQuestParticipant` / `RemovePartyQuestParticipant` | Manage a party quest's roster. |
| `CompleteQuest(Player, Quest)` | Manually complete a quest that's ready to turn in. Routes by sharing mode automatically. |
| `FailQuest(Player, Quest)` | Fail a quest. |
| `AbandonQuest(Player, Quest)` | Player-initiated cancel (requires `bCanAbandon`). |

## Event notifications

See [04 — Event-Driven Progress](04-Event-Driven-Progress.md) for the full matching rules.

| Function | Signature |
|---|---|
| `NotifyQuestEvent` | `(Player, EventID, TargetIdentifier, Count = 1)` |
| `NotifyKillEvent` | `(Player, EnemyTag, Count = 1)` |
| `NotifyCollectEvent` | `(Player, ItemID, Count = 1)` |
| `NotifyInteractionEvent` | `(Player, TargetID)` |
| `NotifyLocationReachedEvent` | `(Player, LocationID)` |
| `NotifyCustomQuestEvent` | `(Player, EventID, TargetIdentifier, Count = 1)` |

## Quest queries

| Function | Returns |
|---|---|
| `GetAvailableQuests(Player, AllQuests)` | Quests from `AllQuests` the player can currently start. |
| `IsQuestActive(Player, Quest)` / `IsQuestCompleted(Player, Quest)` | Status checks, routed by sharing mode. |
| `CanStartQuest(Player, Quest)` | Prerequisites met + not already active/completed. |
| `IsTargetRelevant(Player, EventID, TargetID)` | `true` if the player has an active objective that a `NotifyQuestEvent(EventID, TargetID)` would advance (checks own + party managers). Drives world-target markers. |
| `GetActiveQuests(Player)` | The player's active Personal + Individual quests. |
| `GetActiveSharedQuests(WorldContextObject)` | Active Shared quests from the `GameState`. |
| `GetCompletedQuests(Player)` | The player's completed quests. |
| `GetFailedQuests(Player)` | The player's failed quests (time expired, manually failed, etc.). |
| `GetQuestProgress(Player, Quest, OutObjectives, OutProgress)` | Parallel arrays of every objective and its current progress. |
| `GetQuestCompletionPercentage(Player, Quest)` | `0.0`–`1.0`, based on required objectives only. |
| `AreAllRequiredObjectivesComplete(Player, Quest)` | Same check `IsQuestReadyToTurnIn` uses internally. |
| `GetQuestTimeRemaining(Player, Quest)` | Seconds left on a timed quest (`0` if not timed/active). |
| `GetQuestTimeRemainingText(Player, Quest)` | The above, formatted `MM:SS`. |
| `GetObjectiveTimeRemaining(Player, Quest, Objective)` | Seconds left on a timed *objective* — independent of the quest's own timer (see [03](03-Authoring-Quests.md#timed-objectives)). |
| `GetObjectiveTimeRemainingText(Player, Quest, Objective)` | The above, formatted `MM:SS`. |

## Progress utilities (stateless)

Useful for UI code that only has an `FDemoQuestObjectiveProgress` or an enum value in hand, with no manager lookup needed:

| Function | Returns |
|---|---|
| `GetObjectiveProgressText(Progress)` | `"3/5"` style text. |
| `IsObjectiveCompleted(Progress)` | `bool`. |
| `GetObjectiveProgressPercentage(Progress)` | `0.0`–`1.0`. |
| `GetQuestStateText(State)` | Localized-ready text for `EDemoQuestState`. |
| `GetObjectiveStateText(State)` | Localized-ready text for `EDemoQuestObjectiveState`. |

## Party quest utilities

| Function | Purpose |
|---|---|
| `GetPartyQuestParticipants(WorldContextObject, Quest)` | Roster for a Shared or Individual quest. |
| `GetPlayerContribution(WorldContextObject, Quest, Player)` | A player's share of a Shared quest's progress. |
| `IsPlayerParticipant(WorldContextObject, Quest, Player)` | Roster membership check. |
| `HasPlayerCompletedIndividualQuest(Player, Quest)` | Individual-quest personal completion check. |

## Where to go next

- The lower-level `UDemoQuestManagerComponent` API these functions wrap (delegates, party-quest internals): [01](01-Core-Concepts.md), [04](04-Event-Driven-Progress.md), [06](06-Multiplayer.md)
- Console commands built on the same queries: [09 — Validation & Cheats](09-Validation-And-Cheats.md)


---
*Generated 2026-08-04 14:54 UTC from `Docs/Full/` - do not edit this page directly.*
