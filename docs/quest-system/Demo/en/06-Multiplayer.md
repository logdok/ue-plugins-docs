# 06 — Multiplayer

*🇬🇧 English | [🇺🇦 Українська](../uk/06-Multiplayer.md)*

Multiplayer support is not bolted on — it is the reason the plugin is shaped the way it is (see [01 — Core Concepts](01-Core-Concepts.md) for the one-component-two-locations model). This chapter covers the party-quest API and the pattern that lets the *same* map and the *same* content work in single-player and in multiplayer.

## Server authority

Every mutating method on `UDemoQuestManagerComponent` (`StartQuest`, `CompleteQuest`, `FailQuest`, `AbandonQuest`, `UpdateObjectiveProgress`, `SetObjectiveProgress`, `NotifyQuestEvent`, `AddParticipant`, `RemoveParticipant`, …) checks authority internally and forwards to the server when called from a client. You never need to think about this — call the public method from anywhere, client or server, and it does the right thing, **including when called on the party manager for a Shared/Individual quest**: `GameState` has no single owning client connection, so a reliable RPC can't be declared directly on a `GameState`-hosted component (the engine silently refuses to execute it for any client) — these calls are relayed through the local player's own manager under the hood instead. Purely an implementation detail; nothing you need to set up.

One caveat worth knowing: on a client, these methods return `true` optimistically once the request has been sent, not once the server has actually accepted it. Treat the delegates (`OnQuestStateChanged`, `OnQuestCompleted`, …) or a later state check as the real source of truth if a call could plausibly be rejected server-side — don't drive UI purely off the return value.

`ActiveQuests` replicates automatically (`OnRep_ActiveQuests`); `CompletedQuests`/`FailedQuests`/`PartyQuestStates` replicate too. All of these replicate to every observer of the host actor — so on a `PlayerState` manager a player's own quest data reaches all clients, not just the owner. That's because the single component class also hosts collective Shared progress on the owner-less `GameState`, which rules out a blanket `COND_OwnerOnly`. Treat other players' entries as incidental/read-only; if you need strict owner-only scoping, split into two component subclasses.

## Starting a party quest

Shared and Individual quests are started with a **participant list**, not a single player:

```cpp
// Explicit participant list
UDemoQuestManagerComponent* PartyManager = UDemoQuestBlueprintLibrary::GetPartyQuestManager(this);
PartyManager->StartPartyQuest(QuestData, { Player1State, Player2State });

// Or, via the Blueprint Library (works from anywhere with a world context):
UDemoQuestBlueprintLibrary::StartPartyQuest(this, QuestData, Participants);
```

### The single-player + multiplayer pattern

`StartPartyQuestForAllPlayers()` starts a party quest with **whoever is currently connected**:

```cpp
UDemoQuestBlueprintLibrary::StartPartyQuestForAllPlayers(this, SharedQuestData);
```

In standalone play, that is a party of one — the quest behaves exactly like a Personal quest to that one player. On a server with several players connected, the same call enrolls all of them. This is exactly how `UDemoQuestGiverComponent::GiveQuest()` behaves for Shared/Individual quests, and it's why a `UDemoQuestInteractorComponent` interacting with a giver "just works" in both modes without the level designer doing anything different.

Combine this with `bAutoEnrollNewParticipants` (on the quest asset, see [03](03-Authoring-Quests.md)) to also pick up players who join **after** the quest already started — `UDemoQuestWorldSubsystem` calls `EnrollPlayerInAutoJoinQuests()` automatically for every new `PlayerState`, so drop-in co-op works with zero extra code.

`MinPartySize` / `MaxPartySize` on the quest asset are enforced by `StartPartyQuest` — use `MinPartySize` to force a quest to require real co-op even though the API would happily start it solo.

## Managing participants

```cpp
PartyManager->AddParticipant(QuestData, NewPlayerState);
PartyManager->RemoveParticipant(QuestData, LeavingPlayerState);   // fails a Shared quest with 0 participants left

TArray<APlayerState*> Participants = PartyManager->GetParticipants(QuestData);
bool bIn = PartyManager->IsParticipant(QuestData, SomePlayer);
```

## Shared quests: collective progress + contribution

A Shared quest has exactly one set of objective counters, replicated from the `GameState`. Every participant's events add to the *same* counters (subject to the identifier-matching rules in [04](04-Event-Driven-Progress.md)), while each player's individual share is tracked separately for leaderboards or bonus rewards:

```cpp
int32 MyKills = PartyManager->GetPlayerContribution(QuestData, MyPlayerState);
```

## Individual quests: separate progress, shared completion gate

Each participant gets their own copy of the objectives on their own `PlayerState` — `AddParticipant` on an Individual quest also calls `StartQuest` for that player. The `GameState` only tracks who has finished:

```cpp
bool bDone = UDemoQuestBlueprintLibrary::HasPlayerCompletedIndividualQuest(MyPlayerState, QuestData);
```

Internally, when a player finishes their own copy, `MarkParticipantCompleted()` records it on the party manager; once every participant is marked complete, the party-side bookkeeping for that quest is cleared automatically. If a player disconnects with no party manager available (rare, purely-local setups), the quest still completes locally for that player — see [04](04-Event-Driven-Progress.md) for how routing falls back gracefully.

## Where to go next

- Persisting quest progress across sessions (including what does *not* survive — party rosters): [07 — Serialization](07-Serialization.md)

<!-- doc-footer:start -->
---
*Generated 2026-08-05 11:44 UTC from `Docs/Full/` - do not edit this page directly.*
<!-- doc-footer:end -->
