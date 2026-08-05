# QuestSystemDemo User Guide

*🇬🇧 English | [🇺🇦 Українська](../uk/README.md)*

This is the full reference guide for the QuestSystemDemo plugin — an event-driven, data-driven quest system for **Unreal Engine 5.7** with first-class multiplayer support and zero-config integration. Designers author quests as Data Assets; gameplay code reports a handful of events; the plugin does the rest, including the network replication.

## Quick Start

1. Copy `Plugins/QuestSystemDemo` into your project's `Plugins/` folder.
2. Enable it: **Edit > Plugins > QuestSystemDemo**.
3. That's it — no code required yet. `UDemoQuestWorldSubsystem` automatically attaches a quest manager to every player and to the game state.

Create your first quest:

1. In the Content Browser: **Add > Miscellaneous > Data Asset**, pick **Quest Objective Data**. Set `ObjectiveType`, `TargetCount`, and `ObjectiveIdentifier` (e.g. `KillTarget` / `3` / `"Bandit"`).
2. Add another Data Asset, this time **Quest Data**. Give it a `QuestName` and add your objective to `Objectives`.
3. Add a **Quest Giver Component** to any Actor in your level and drop your quest into `AvailableQuests`.
4. Add a **Quest Interactor Component** to your player pawn and bind an input to `TryInteract`.
5. Press Play, walk up to the actor, interact — the quest starts. Report progress from your own gameplay code:

```cpp
// e.g. in your damage-handling code, once an enemy dies:
UDemoQuestBlueprintLibrary::NotifyKillEvent(PlayerState, "Bandit");
```

The objective updates, the quest completes, and `GiveQuestRewards()` fires — override it (in Blueprint or C++) to hand out your own rewards. The rest of this guide covers everything above in depth, plus multiplayer, save/load, the full Blueprint API, and C++ extension points.

## Who should read what

- **Designers** authoring quest content: read [01](01-Core-Concepts.md), [03](03-Authoring-Quests.md), and [05](05-World-Components.md).
- **Gameplay programmers** wiring up events and rewards: read [02](02-Zero-Config-Integration.md), [04](04-Event-Driven-Progress.md), and [10](10-CPP-Integration.md).
- **Multiplayer programmers**: read [06](06-Multiplayer.md) after [01](01-Core-Concepts.md).
- **UI programmers**: read [08](08-Blueprint-Library-Reference.md).
- **Tooling / QA**: read [09](09-Validation-And-Cheats.md).

In a hurry, or something's behaving oddly? Start at [11 — FAQ](11-FAQ.md).

## Chapters

| # | Chapter | What's in it |
|---|---------|---------------|
| 00 | [Demo Limitations](00-Demo-Limitations.md) | What the demo limits, what it does not, and what to expect when you switch |
| 01 | [Core Concepts](01-Core-Concepts.md) | Quests, objectives, states, sharing modes, the one-component-two-locations model |
| 02 | [Zero-Config Integration](02-Zero-Config-Integration.md) | `UDemoQuestWorldSubsystem`, `UDemoQuestSystemSettings`, getting a quest manager |
| 03 | [Authoring Quests](03-Authoring-Quests.md) | Full `UDemoQuestData` / `UDemoQuestObjectiveData` property reference |
| 04 | [Event-Driven Progress](04-Event-Driven-Progress.md) | `NotifyQuestEvent`, matching rules, sequential objectives, shared kill-credit |
| 05 | [World Components](05-World-Components.md) | Giver, Receiver, the marker subclasses, Interactor, Quest Target, Location Trigger, ready-made actors |
| 06 | [Multiplayer](06-Multiplayer.md) | Replication, party quests, single-player + multiplayer on one map |
| 07 | [Serialization](07-Serialization.md) | `ExportState` / `ImportState`, embedding in your save game |
| 08 | [Blueprint Library Reference](08-Blueprint-Library-Reference.md) | Full `UDemoQuestBlueprintLibrary` function reference |
| 09 | [Validation & Cheats](09-Validation-And-Cheats.md) | Circular-dependency detection, console commands |
| 10 | [C++ Integration](10-CPP-Integration.md) | Custom rewards, extending components, sending events from gameplay code |
| 11 | [FAQ](11-FAQ.md) | Short answers to the common questions, and the "why isn't this working?" checklist |

## Conventions used in this guide

- `UDemoQuestData` refers to a quest asset; `UDemoQuestObjectiveData` refers to an objective asset. Both are `UPrimaryDataAsset`s you create in the Content Browser.
- Code snippets are C++ unless stated otherwise; every function shown is also callable from Blueprint.
- "The manager" means `UDemoQuestManagerComponent` — see [01](01-Core-Concepts.md) for where it lives and what it does.

<!-- doc-footer:start -->
---
*Generated 2026-08-05 12:24 UTC from `Docs/Full/` - do not edit this page directly.*
<!-- doc-footer:end -->
