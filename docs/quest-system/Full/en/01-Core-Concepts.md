# 01 — Core Concepts

*🇬🇧 English | [🇺🇦 Українська](../uk/01-Core-Concepts.md)*

## Quests and objectives are data

A **quest** (`UQuestData`) is a `UPrimaryDataAsset` you create in the Content Browser: a name, a description, a list of objectives, optional prerequisites, and rewards. An **objective** (`UQuestObjectiveData`) is its own data asset: what the player must do, how many times, and an identifier used to match it against gameplay events.

Nothing about a quest is hardcoded. Designers build content entirely as assets; see [03 — Authoring Quests](03-Authoring-Quests.md) for the full property reference.

## Quest lifecycle

A quest moves through these states (`EQuestState`):

| State | Meaning |
|-------|---------|
| `Locked` | Exists, but prerequisites are not met. |
| `Available` | Prerequisites are met; the player can start it. |
| `Active` | In progress. |
| `Completed` | Finished — rewards were given. |
| `Failed` | Failed (time ran out, or the game explicitly failed it). |

Each objective inside an active quest has its own state (`EQuestObjectiveState`): `Inactive` (not started yet — used for sequential objectives), `Active` (currently trackable), `Completed`, or `Failed`.

`IsQuestReadyToTurnIn()` is a different question from `IsQuestCompleted()`: the former is true once every required objective is done (turn-in prompt should show); the latter is true only after `CompleteQuest()` has actually run and rewards were given.

## One component, two locations

There is a single component class, `UQuestManagerComponent`. What it manages depends entirely on **where it is attached**:

- On a **`PlayerState`** → manages that player's **Personal** and **Individual** quests.
- On the **`GameState`** → manages **Shared** quests for the whole session (the "party manager").

In both locations the quest arrays replicate to **every** observer of the host actor — a player's own quests on a `PlayerState` reach all clients, not just the owner. That is a deliberate consequence of one class serving both locations; see [06 — Multiplayer](06-Multiplayer.md#server-authority) for why owner-only scoping isn't possible here and what to do if you need it.

You never need to create these components yourself — see [02 — Zero-Config Integration](02-Zero-Config-Integration.md). `IsPersonalQuestManager()` / `IsPartyQuestManager()` tell you which one you have.

## Sharing modes

Every quest has a `QuestSharingMode` (`EQuestSharingMode`), and it decides everything about how the quest behaves in multiplayer:

| Mode | Where progress lives | Behavior |
|------|----------------------|----------|
| **Personal** | The player's own `PlayerState` | A solo quest. Each player who starts it gets their own independent copy; other players don't see it and can't contribute. |
| **Shared** | The `GameState` | One collective objective count for the whole party. Every participant's progress adds to the same number. Individual contribution is still tracked (for leaderboards, bonus rewards, etc.). |
| **Individual** | Each player's own `PlayerState` | Every participant tracks their own copy of the objectives. The `GameState` only keeps the roster and knows when *everyone* has finished. |

### Do I need Shared to support both single-player and multiplayer?

**No — all three modes work whether the level is played solo or with several players.** `QuestSharingMode` does not decide *whether* a quest works in multiplayer; it decides *how* it behaves once more than one player is present:

- **Personal** works fine in multiplayer too — it's just never collective. Each player who talks to the giver gets their own independent copy, with zero interaction between players. In a solo session this is indistinguishable from "the only player"; in a multiplayer session it's just several players doing the same quest in parallel, unaware of each other.
- **Shared** and **Individual** are what you reach for when you specifically want *one instance that automatically scales to the party* — started via `StartPartyQuestForAllPlayers()` (see [06 — Multiplayer](06-Multiplayer.md)), which enrolls "whoever is currently connected." In a solo session that's a party of one; nothing about the quest or the level needs to change for multiplayer.

So the real decision isn't "which mode lets this run in both modes" (all of them do) — it's "does progress belong to one collective counter (**Shared**), to each player separately with a shared finish line (**Individual**), or fully independently per player (**Personal**)". `DemoMap` deliberately mixes all three on one level to show the difference side by side — open it in the editor and compare `DQ_CrystalWarmup` (Personal) against `DQ_PartyHarvest` (Shared) and `DQ_Pilgrimage` (Individual).

## Where to go next

- Setting up the plugin with zero code: [02 — Zero-Config Integration](02-Zero-Config-Integration.md)
- Building your first quest asset: [03 — Authoring Quests](03-Authoring-Quests.md)

<!-- doc-footer:start -->
---
*Last updated: 2026-08-04 17:36 UTC*
<!-- doc-footer:end -->
