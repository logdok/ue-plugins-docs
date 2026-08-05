# Demo Limitations

*🇬🇧 English | [🇺🇦 Українська](../uk/00-Demo-Limitations.md)*

This is the **demo edition** of QuestSystem. It exists so you can evaluate the plugin properly: nothing is stubbed out, no feature is hidden behind the paywall, and no chapter of this guide describes something the demo cannot do. What the demo does have is a budget, a watermark, and a refusal to run in a shipped game.

## 1. Three quests per play session

A play session may **start** up to **3 distinct quests**. The counter is keyed by `QuestID` (the asset name when `QuestID` is unset) and resets every time a game world begins play — press Play again and you get a fresh three.

What this means in practice:

- Re-taking a quest you already started (after abandoning or resetting it) is **free** — it already holds its slot.
- Three slots is deliberately enough to see the whole system at once: one **Personal**, one **Shared** and one **Individual** quest running side by side, or a three-step prerequisite chain A → B → C.
- Loading a saved game claims slots like anything else — `ImportState` restores quests up to the same budget, it is not a way around it.
- When the budget is exhausted, `StartQuest` / `StartPartyQuest` return `false`, log a warning, and print an on-screen message naming the quest that was blocked. Nothing crashes and nothing is corrupted; the quest simply does not start.

The full plugin has no such counter.

## 2. On-screen watermark

While a game world is running, the demo draws a line in the corner of the screen:

```
QUEST SYSTEM - DEMO EDITION - 2/3 quests used this session
```

It doubles as the live budget readout. The debug overlay (`J` in the demo project, or the `DemoQuestOverlay` console command) also states the limit under its header.

## 3. No Shipping builds

In a `Shipping` configuration the plugin disables itself: `UDemoQuestWorldSubsystem` is never created, no quest managers are attached, and no quest can be started. Development and DebugGame builds — including packaged ones — work normally, so you can evaluate the plugin outside the editor.

## Installed next to the full plugin

Both editions can be enabled in the same project — that is what the `Demo` naming is for. By default the demo then **stands down**: with the full plugin's module loaded it attaches no quest managers, draws no watermark and starts no quests, logging one line to say so. That keeps a project that already owns the real plugin free of demo noise.

To run both at once anyway, uncheck **Stand Down When Full Plugin Present** in **Project Settings → Game → Quest System (Demo)**.

## What is *not* limited

Everything else, specifically:

- all three sharing modes (Personal / Shared / Individual) and the full party-quest system, including late-join enrolment and contribution tracking;
- unlimited objectives per quest, all objective types including `Custom`, sequential objectives, optional objectives, quest- and objective-level timers;
- every world component: giver, receiver, interactor, quest target, location trigger, and all marker subclasses;
- prerequisites and the circular-dependency validator;
- `ExportState` / `ImportState` and the finished-quest snapshots;
- the debug overlay, every console command, and the whole Blueprint API;
- multiplayer replication, with any number of players.

## Two things to know before you build on it

**Assets do not transfer to the full plugin.** So that the two editions can be installed side by side, every UObject type in the demo is renamed (`UQuestData` → `UDemoQuestData`, and so on for every class, struct and enum). Unreal identifies asset classes by those names, so a quest you author here cannot be opened by the full plugin. Content made while evaluating is throwaway content.

**Names differ from the documentation of the full plugin.** Everything in this guide is already written in demo naming, so it matches your editor. When you move to the full plugin, drop the `Demo` marker: `UDemoQuestManagerComponent` → `UQuestManagerComponent`, `DemoQuestStart` → `QuestStart`, module `QuestSystemDemo` → `QuestSystem`.

## Getting the full plugin

The full edition removes the quest budget and the watermark, ships in `Shipping` builds, and adds the automation test suite. Everything you learned evaluating the demo carries over unchanged — the API is identical apart from the naming marker. Its documentation — the same guide you're reading now, without the `Demo` prefixes — is [Docs/Full/en](../../Full/en/README.md).

<!-- doc-footer:start -->
---
*Generated 2026-08-05 11:23 UTC from `Docs/Full/` - do not edit this page directly.*
<!-- doc-footer:end -->
