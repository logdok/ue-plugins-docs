# 04 — Event-Driven Progress

*🇬🇧 English | [🇺🇦 Українська](../uk/04-Event-Driven-Progress.md)*

Gameplay code never touches objective counters directly. It reports that *something happened*, and the quest system figures out which active objectives care.

## Reporting an event

The universal entry point is `NotifyQuestEvent`, available on the manager component and, more conveniently, as static functions on `UQuestBlueprintLibrary` that take a `APlayerState*` and route to the right manager for you:

```cpp
// Player killed a bandit
UQuestBlueprintLibrary::NotifyKillEvent(PlayerState, "Bandit");

// Player collected 3 apples
UQuestBlueprintLibrary::NotifyCollectEvent(PlayerState, "Apple", 3);

// Player reached a checkpoint
UQuestBlueprintLibrary::NotifyLocationReachedEvent(PlayerState, "ForestShrine");

// Player interacted with an NPC
UQuestBlueprintLibrary::NotifyInteractionEvent(PlayerState, "VillageElder");

// Anything else (crafting, reputation, PvP...)
UQuestBlueprintLibrary::NotifyCustomQuestEvent(PlayerState, "Craft", "HealthPotion", 1);

// Or, for full control, the general form used by all of the above:
UQuestBlueprintLibrary::NotifyQuestEvent(PlayerState, EventID, TargetIdentifier, Count);
```

These are safe to call from client or server code — internally they forward to the server via RPC when needed. `Count` defaults to `1` and can be higher (e.g. picking up a stack of 5 apples at once).

Canonical event IDs are also available as C++ constants, so you don't have to spell string literals: `QuestEvents::Kill`, `QuestEvents::Collect`, `QuestEvents::Location`, `QuestEvents::Interact` (in `Core/QuestTypes.h`).

## How matching works

An event matches an objective when **both** are true:

1. `Objective->ObjectiveIdentifier == TargetIdentifier` (exact `FName` match).
2. The event ID is compatible with `Objective->ObjectiveType` — see the table in [03 — Authoring Quests](03-Authoring-Quests.md). `Custom` objectives additionally require the event ID to equal `ExpectedEventID` (unless `ExpectedEventID` is unset).

Every objective in every currently **active** quest handled by that manager is checked. A single event can therefore advance more than one objective at once, and can advance more than one quest at once.

## ⚠ Shared credit across quests with the same identifier

This is worth calling out explicitly, because it surprises people the first time they see it: matching happens by `(ObjectiveType, ObjectiveIdentifier)` only — **it does not know or care which quest "owns" a target**. If two different active quests (or two objectives inside the same quest) both define `KillTarget` / `"Bandit"`, killing **one** bandit sends **one** event, and that one event credits **both** objectives.

```
Quest A: "Bandit Threat"      — Defeat Bandits  0/5
Quest B: "Bandit Threat 2222" — Defeat Bandits  0/2

NotifyKillEvent(PlayerState, "Bandit")   // one kill

Quest A: Defeat Bandits  1/5   (+1)
Quest B: Defeat Bandits  1/2   (+1)   <- both credited from the same kill
```

**This is intentional, not a bug.** It is the same "shared kill credit" behavior most games with concurrent kill-quests use (e.g. two active quests that both want you to kill wolves): the world doesn't have a separate copy of each enemy per quest, so one kill naturally counts toward every matching quest that is currently active. If your design calls for exclusive crediting instead (a kill should only ever progress *one* of several matching quests), that is a deliberate policy choice you implement yourself — it requires deciding which quest "wins" when several match, which this plugin does not decide for you.

The scope of this sharing is **per manager**: a player's own Personal + Individual quests share credit with each other (they live on the same `PlayerState` manager); Shared quests share credit with each other separately, on the `GameState` manager. A Personal quest on one player never affects another player's quests.

## Sequential objectives

An objective with `bIsSequential = true` starts `Inactive` and only becomes `Active` once every earlier *required* sequential objective in the same quest has completed — this lets you build multi-step quests ("Step 1 complete. Now do Step 2.") without any extra code. `OnSequentialObjectiveActivated` fires when a step unlocks; use it to update a quest tracker.

Non-sequential objectives are all `Active` from the moment the quest starts and can be worked on in any order.

## Setting progress directly

Most objectives are incremented by events, but two lower-level functions are available when you need exact control:

```cpp
// Add to the current value (clamped to TargetCount)
QuestManager->UpdateObjectiveProgress(QuestData, Objective, /*Amount*/ 1);

// Set an exact value (clamped to [0, TargetCount])
QuestManager->SetObjectiveProgress(QuestData, Objective, /*NewProgress*/ 5);
```

Both exist on `UQuestManagerComponent` and route through the server automatically, just like `NotifyQuestEvent`.

## Listening for progress

Bind to the manager's delegates instead of polling:

| Delegate | Fires when |
|---|---|
| `OnQuestStarted(Quest)` | A quest starts. |
| `OnQuestStateChanged(Quest, NewState)` | Any state transition. |
| `OnObjectiveUpdated(Quest, Objective, Progress)` | An objective's counter changes. |
| `OnObjectiveCompleted(Quest, Objective)` | An objective finishes (fires once). |
| `OnSequentialObjectiveActivated(Quest, Objective)` | A sequential step unlocks. |
| `OnQuestCompleted(Quest, Rewards)` | The quest completes — **after** rewards were already given; use this for UI/FX only. |
| `OnQuestFailed(Quest)` | The quest fails (timer expiry or explicit `FailQuest`). |
| `OnQuestAvailable(Quest)` | A quest's prerequisites just became satisfied (via `CheckAndNotifyAvailableQuests`). |
| `OnQuestTimerWarning(Quest, TimeRemaining)` | A timed quest crosses its own warning threshold (`UQuestData::TimerWarningThreshold`). |
| `OnObjectiveTimerWarning(Quest, Objective, TimeRemaining)` | A timed objective crosses its own warning threshold (`UQuestObjectiveData::TimerWarningThreshold`) — independent of the quest-level one. |
| `OnQuestEventNotified(EventID, TargetIdentifier, Count, Player, bMatchedAnyObjective)` | Every `NotifyQuestEvent`/`NotifyPartyQuestEvent` call this manager processes — **including ones that match nothing** (except on the party manager when the player has no active Shared quest to check at all — see below). |

`OnQuestEventNotified` is different in kind from the others above: it is not a gameplay reaction hook, it is a diagnostic one — it fires for *every* event this manager sees, whether or not `bMatchedAnyObjective` ends up true, which makes it the fastest way to answer "why didn't my quest progress?" (typo'd `TargetIdentifier`, wrong `EventType`, objective not yet `Active`, …). One exception, on the party manager only: `NotifyQuestEvent` forwards every event there unconditionally in case a Shared quest cares, so a player with no active Shared-quest participation would otherwise produce a permanently, meaninglessly "unmatched" broadcast for every single event — `NotifyPartyQuestEvent` skips the broadcast entirely in that case instead. The `QuestSystemDebug` module's overlay (`J` in the demo) binds to it — alongside `OnQuestStateChanged` and `OnObjectiveUpdated` — to drive its live **Events** tab — see [09 — Validation & Cheats](09-Validation-And-Cheats.md).

### Binding to a delegate in Blueprint

Every delegate above is a plain `BlueprintAssignable` member, so binding to one is the standard Unreal pattern — no plugin-specific setup:

1. Get a reference to the manager you care about: `Get Quest Manager (Player)` for a player's own Personal/Individual quests, or `Get Party Quest Manager (World Context)` for Shared quests (both on `Quest Blueprint Library`).
2. Drag off the returned component reference pin, start typing the delegate's name (e.g. **On Objective Completed**), and pick **Bind Event to On Objective Completed**.
3. The editor creates a matching `Custom Event` node with the delegate's parameters as output pins (`Quest`, `Objective`, …) — wire your reaction from there (play a sound, flash a tracker entry, update a widget).

A widget Blueprint showing a live quest tracker, or a Character/PlayerController Blueprint reacting to progress with VFX/SFX, are the two most common places to do this.

**Timing tip:** on a network client, `PlayerState`/`GameState` (and therefore the quest manager living on them) can replicate in *after* `BeginPlay` runs, so a bind attempted too early silently does nothing — the reference is still null at that point. Guard the bind with a null check and retry shortly after (a `Delay` node, or re-attempt in `OnRep_PlayerState`) rather than assuming `BeginPlay` is late enough.

## Where to go next

- Placing NPCs, boards, and chests that call these functions for you: [05 — World Components](05-World-Components.md)
- The multiplayer routing rules behind "which manager handles this event": [06 — Multiplayer](06-Multiplayer.md)

<!-- doc-footer:start -->
---
*Last updated: 2026-08-04 15:08 UTC*
<!-- doc-footer:end -->
