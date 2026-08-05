# 11 — FAQ

*🇬🇧 English | [🇺🇦 Українська](../uk/11-FAQ.md)*

Short answers to the questions that come up most, with links to the chapter that covers each in depth. If something here contradicts a chapter, the chapter is right — tell us, and we'll fix this page.

- [Getting started](#getting-started)
- [Choosing a sharing mode](#choosing-a-sharing-mode)
- [Authoring quests](#authoring-quests)
- [Why isn't it working?](#why-isnt-it-working)
- [UI and presentation](#ui-and-presentation)
- [Multiplayer](#multiplayer)
- [Shipping and packaging](#shipping-and-packaging)
- [Extending the plugin](#extending-the-plugin)

## Getting started

### Do I have to subclass `PlayerState`, `GameState` or `GameMode` to use this?

No. That's the whole point of the zero-config design: `UDemoQuestWorldSubsystem` attaches a `UDemoQuestManagerComponent` to every `PlayerState` and to the `GameState` at runtime, whatever classes your project already uses. Drop the plugin in, enable it, and the API works. See [02 — Zero-Config Integration](02-Zero-Config-Integration.md).

If you'd rather place the component yourself (on your own `PlayerState` subclass, say), do that — the subsystem detects an existing component and leaves it alone. You can also turn auto-creation off entirely with `bAutoCreatePlayerQuestManagers` / `bAutoCreatePartyQuestManager` in Project Settings → Game → Quest System (Demo).

### Do I need to write C++?

No. Every function in [`UDemoQuestBlueprintLibrary`](08-Blueprint-Library-Reference.md) is Blueprint-callable, quests are Data Assets, and the world components are drop-on-any-Actor. Even reward granting is a `BlueprintNativeEvent` — you can override `GiveQuestRewards` in a Blueprint subclass of `UDemoQuestManagerComponent` and point `QuestManagerClass` at it, no C++ file involved. [10 — C++ Integration](10-CPP-Integration.md) is there if you *want* C++, not because you need it.

### Where do quest assets have to live?

Anywhere in your content. Runtime lookup scans the Asset Registry by class, so folder layout is yours to choose and moving assets never breaks anything. **One exception that matters:** packaged builds still need the `AssetManager` "Quest" entry in `DefaultGame.ini` pointing at your real quest folders, or the quests won't be cooked into the build. See [09 — Validation & Cheats](09-Validation-And-Cheats.md).

### Does any of this work in single-player?

Yes — and not as an afterthought. All three sharing modes work standalone; a "party" quest in a single-player game is simply a party of one. That's deliberate, so the same map and the same quest assets work in both modes without a separate content path. See [06 — Multiplayer](06-Multiplayer.md).

### Can I add this to a project that's already well underway?

Yes. It doesn't require a specific base class, a specific input system, or a particular project layout, and it adds no gameplay of its own until you author a quest. The one thing to plan for is where your gameplay code will report events from — see [04 — Event-Driven Progress](04-Event-Driven-Progress.md).

## Choosing a sharing mode

### Personal, Shared, Individual — which one do I want?

- **Personal** — a solo quest. Progress lives on that one player. Use this unless you have a reason not to.
- **Shared** — one collective progress pool for the whole party. Ten bandits between everyone, not each. Each player's contribution is tracked separately for rewards or a scoreboard.
- **Individual** — everyone gets their own copy of the objectives, and the quest is "done for the party" only once every participant has finished their own. Ten bandits *each*.

[01 — Core Concepts](01-Core-Concepts.md) has the full model, including where each one's data physically lives.

### Can the same quest be Personal for one player and Shared for another?

No — `QuestSharingMode` is a property of the quest asset, so it's fixed for everyone. If you need both behaviors, make two assets.

### Which mode should I pick if I'm only making a single-player game?

Personal. The other two exist to coordinate several players; with one player, Individual behaves like Personal with extra bookkeeping, and Shared moves the progress onto the GameState manager for no benefit.

## Authoring quests

### How do I make a chain where one quest unlocks the next?

Put the earlier quest in the later quest's `PrerequisiteQuests`. `CanStartQuest` then reports the later one as unavailable until the prerequisite is completed. The plugin validates the whole graph for circular dependencies — in the editor when you save, on world start in development builds, and on demand via `DemoQuestValidate`. See [09 — Validation & Cheats](09-Validation-And-Cheats.md).

### How do I make objectives happen in a fixed order?

Tick `bIsSequential` on the objectives that should wait their turn. A sequential objective stays inactive until the previous one is completed, and only an **Active** objective accepts progress — which is exactly what makes "go here, *then* kill that" work. See [03 — Authoring Quests](03-Authoring-Quests.md).

### How do I make an objective the player can skip?

`bIsOptional`. Optional objectives never block completion and never fail the quest — including when a timed optional objective runs out, which just marks that one objective failed and leaves the quest running. See [03 — Authoring Quests](03-Authoring-Quests.md).

### My quest needs something the built-in objective types don't cover.

Use `EDemoQuestObjectiveType::Custom` with an `ExpectedEventID`, then report it with `NotifyCustomQuestEvent`. That covers "craft 3 swords", "win 2 races", "survive 5 waves" — anything your game can detect and name. See [04 — Event-Driven Progress](04-Event-Driven-Progress.md).

### Can I attach my own data to a quest without modifying the plugin?

Yes — `CustomData` (a `TMap<FName, FString>`) exists on both quest and objective assets exactly for that. The property set is deliberately lean and genre-neutral; `CustomData` plus the free-form `QuestCategory` tag is the intended escape hatch instead of the plugin growing a field for every game's needs.

### Should the quest complete on its own, or when the player turns it in?

`bAutoComplete` on the quest decides. Leave it off and the quest sits at "ready to turn in" until something calls `CompleteQuest` — typically a `UDemoQuestReceiverComponent` on a turn-in NPC. Turn it on for fire-and-forget quests with no turn-in step.

## Why isn't it working?

### My Shared quest never shows as available/active/completed. No errors anywhere.

This is the single most common mistake with the plugin, and it's silent by design rather than by accident. A Shared quest's progress lives on the **GameState** manager, not the player's — so `GetQuestManager(Player)` returns a manager that has simply never heard of that quest, and every query politely answers "no".

Use `GetQuestManagerForQuest(Player, Quest)`, which routes by sharing mode for you, or check both managers when you're aggregating a list. See [08 — Blueprint Library Reference](08-Blueprint-Library-Reference.md).

### I report an event and nothing happens.

Work down this list — the first four cover almost every case:

1. **Is the quest actually active?** Events only advance active quests.
2. **Does `TargetIdentifier` match `ObjectiveIdentifier`?** Typos and stray whitespace are the usual culprits. Capitalization is *not* — these are `FName`s, and `FName` comparison is case-insensitive, so `"bandit"` and `"Bandit"` do match. Don't waste time chasing case.
3. **Does the event type match the objective type?** `Kill` advances `KillTarget`, `Collect` advances `CollectItem`, and so on. A `Custom` objective additionally has to match its `ExpectedEventID`.
4. **Is the objective Active rather than waiting its turn?** A `bIsSequential` objective ignores progress until the objective before it is done.
5. **Is it a Shared quest being checked on the wrong manager?** See the previous question.

The fastest way to see which of these it is: open the debug overlay's **Events** tab (`DemoQuestOverlay` console command). It lists every event as it arrives and **highlights the ones that matched nothing**, which turns "why didn't my quest progress" into a two-second read. See [09 — Validation & Cheats](09-Validation-And-Cheats.md).

### One kill advanced two different quests at once. Is that a bug?

No, that's intended. Matching is by objective type + identifier, with no notion of "which quest this kill belonged to" — so if two of a player's active quests both want `Kill`/`Bandit`, one bandit credits both. It's the MMO-style behaviour most projects want; when you *don't* want it, give the two objectives different identifiers. See [04 — Event-Driven Progress](04-Event-Driven-Progress.md).

### A player failed a quest and now can't start it again.

That's deliberate: a failed quest stays in the failed list, and `CanStartQuest` refuses it until something calls `DemoQuestReset` for that quest. Otherwise a failed quest would silently re-activate while still flagged failed. Call `DemoQuestReset` (or the console command of the same name) wherever your game decides a second attempt is allowed.

### Everything works in the editor but the quests are missing in a packaged build.

Almost always the cooker: quest assets that nothing references directly won't be cooked unless the `AssetManager` "Quest" primary asset type in `DefaultGame.ini` points at the folders they actually live in. Runtime lookup doesn't need that entry — cooking does. See [09 — Validation & Cheats](09-Validation-And-Cheats.md).

### Do I have to be careful about calling quest functions from a client?

No. Every mutating call checks authority internally and forwards to the server for you, including for Shared quests. The one thing worth knowing is that on a client these functions return `true` optimistically once the request is *sent* — so drive your UI off the delegates or a follow-up state check, not off that return value. See [06 — Multiplayer](06-Multiplayer.md).

## UI and presentation

### Is there a ready-made quest journal or HUD I can ship?

No, and that's on purpose. The plugin ships a **debug overlay** — a dense developer tool showing every quest, filters, live event feed and force-complete buttons — which is built for diagnosing content, not for players. Your quest UI is your game's look and feel, so the plugin gives you the data and stays out of the way.

Build yours from [`UDemoQuestBlueprintLibrary`](08-Blueprint-Library-Reference.md) (`GetActiveQuests`, `GetQuestProgress`, `GetObjectiveProgressText`, …) and bind to the manager's delegates — `OnQuestStateChanged`, `OnObjectiveUpdated`, `OnQuestCompleted`, `OnQuestTimerWarning` and friends — so it updates on change instead of polling.

### How do I show a *completed* quest's objectives in a journal, not just "done"?

Completing a quest discards its live progress, but the plugin keeps a per-objective snapshot for exactly this reason. `GetObjectiveProgress` transparently falls back to it, and `GetFinishedQuestObjectives(Quest)` hands you the whole record, so a history screen can show `3/3` and which optional objectives were actually done — not `0/3`. See [07 — Serialization](07-Serialization.md).

### Are quest texts localizable?

Yes — `QuestName`, `QuestDescription`, `QuestSummary` and the objective texts are `FText`, so they go through Unreal's normal localization pipeline like any other text asset. Identifiers (`ObjectiveIdentifier`, `QuestID`, event IDs) are `FName`s and are **not** for display — never localize those, they're matching keys.

### Can I show a countdown for a timed quest?

`GetQuestTimeRemaining` / `GetQuestTimeRemainingText` for the quest's own deadline, and `GetObjectiveTimeRemaining` / `...Text` for a timed objective's independent one. Both return `0` (or empty text) when there's no timer running, so binding them unconditionally is safe.

## Multiplayer

### Does one player's quest data replicate to other players?

Yes — quest arrays replicate to every observer of their host actor, not just the owning client. That's a consequence of one component class serving both roles: on the owner-less GameState it hosts collective Shared progress, which rules out blanket owner-only replication. Treat other players' entries as incidental and read-only. If you need strict owner-only scoping, that means splitting the component into two subclasses. See [06 — Multiplayer](06-Multiplayer.md).

### What happens if a player joins after a party quest has already started?

If the quest has `bAutoEnrollNewParticipants` (on by default), the subsystem enrolls them automatically as their `PlayerState` appears — drop-in co-op with no extra code. Turn it off for quests that should stay closed to whoever was there at the start.

### Can I require a real party, so a quest can't be soloed?

Set `MinPartySize` on the quest asset. `MaxPartySize` caps the other end. Both are enforced when the party quest starts.

### What happens to a party quest when a player leaves?

Removing a participant drops them from the roster, and for an Individual quest also fails their own copy. If that was the last participant, a Shared quest fails automatically rather than lingering with nobody working on it.

### Is quest progress restored when a player reconnects?

Only what you save and restore yourself — see the serialization question below. Party **rosters** are deliberately not rebuilt automatically: who counts as "the same party" after a disconnect is a game-design decision, not something the plugin should guess. Per-player contributions are preserved. See [07 — Serialization](07-Serialization.md).

## Shipping and packaging

### How do I save and load quest progress?

`ExportState()` gives you a plain struct of asset paths plus progress; `ImportState()` puts it back. It's designed to be embedded in *your* `USaveGame` alongside everything else you persist, rather than the plugin owning a save format you'd have to work around. See [07 — Serialization](07-Serialization.md).

### Will the debug overlay or the cheat commands end up in my shipping build?

No. The cheat commands register only in non-Shipping builds, and the overlay's editor-only affordances (like "Open Asset") compile out of game targets entirely. If you'd rather the developer tooling not be in your build at all, the whole thing lives in a separate `QuestSystemDemoDebug` module you can simply not depend on — core quest logic has no dependency on it.

### Is there anything performance-sensitive I should know about?

Two things, both minor at normal content sizes:

- **Timed quests replicate their countdown.** A quest or objective with a timer updates its remaining time every tick on the server and replicates it. Fine for a handful of timed quests; if you plan to run many at once, deriving the countdown client-side from the start time is the cheaper design.
- **Quest discovery scans the Asset Registry by class.** That's a scan over quest assets, not over your whole content tree, and it's not per-frame work — but cache the result rather than calling `LoadAllQuests` in a tick.

Untimed quests do no per-tick work at all: the manager disables its own tick when no timed quest is active.

## Extending the plugin

### How do I actually give the player their rewards?

Override `GiveQuestRewards` — it's called exactly once per quest, with duplicate protection. The cleanest place is a `UDemoQuestManagerComponent` subclass (Blueprint or C++) set as `QuestManagerClass` in Project Settings, so it applies everywhere without touching any framework class. Don't put reward granting in `OnQuestCompleted`: that delegate fires *after* rewards and has no duplicate protection — it's for UI, audio and analytics. See [10 — C++ Integration](10-CPP-Integration.md).

### How do I make an arbitrary object in my world a quest target?

Add a `UDemoQuestTargetComponent` to it — any actor, not a special base class. It pairs the event to report with a trigger (overlap, damage, interact, destroyed, or your own call) and a response (fire once, respawn, repeat, hide/destroy, spawn an effect). Add one of the marker components too and the actor shows a marker only while a matching objective is actually active. See [05 — World Components](05-World-Components.md).

### Can I use my own marker visuals instead of the built-in ones?

Yes. `UDemoQuestBlueprintMarkerComponent` spawns any actor class you give it — make it a Blueprint with whatever mesh, widget, animation or Niagara effect you like. Implement `IDemoQuestMarkerVisualInterface` on that actor and it also gets told when the marker state changes, so it can react rather than just appear and disappear. Several marker components can coexist on one actor if you want more than one visual. See [05 — World Components](05-World-Components.md).

### Where's the best place to report events from?

Wherever your game already knows the thing happened — the damage handler that concluded an enemy died, the pickup that was collected, the trigger volume that was entered. Call `NotifyKillEvent` / `NotifyCollectEvent` / etc. with the player's `PlayerState` and let the plugin work out which quests care. Quest code doesn't belong in your gameplay systems, and gameplay code shouldn't be poking quest state directly. See [04 — Event-Driven Progress](04-Event-Driven-Progress.md).

## Where to go next

- Still stuck on content behaving oddly? [09 — Validation & Cheats](09-Validation-And-Cheats.md) covers the debug overlay and console commands in full.
- Extending in C++: [10 — C++ Integration](10-CPP-Integration.md).

<!-- doc-footer:start -->
---
*Generated 2026-08-05 14:53 UTC from `Docs/Full/` - do not edit this page directly.*
<!-- doc-footer:end -->
