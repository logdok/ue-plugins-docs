# 09 — Validation & Cheats

*🇬🇧 English | [🇺🇦 Українська](../uk/09-Validation-And-Cheats.md)*

## Dependency validation

`UDemoQuestDependencyValidator` runs a depth-first search over `PrerequisiteQuests` to catch self-dependencies (`A → A`) and circular chains (`A → B → C → A`), both for a single quest and for your whole database:

```cpp
FDemoQuestValidationResult Result;
if (!UDemoQuestValidationLibrary::ValidateAllQuests(AllQuests, Result))
{
    UE_LOG(LogTemp, Error, TEXT("%s"), *Result.ErrorMessage);
    // Result.ProblematicQuest, Result.CircularChain, Result.AllProblematicQuests
}
```

This runs automatically in three places, so most projects never call it directly:

- **Editor-time**, on every `PrerequisiteQuests` edit and on save (`UDemoQuestData::PostEditChangeProperty` / `PreSave` / `IsDataValid` — integrates with Unreal's Data Validation framework).
- **On world start**, via `UDemoQuestWorldSubsystem`, when `bValidateQuestsOnStartup` is enabled (development builds only — see [02](02-Zero-Config-Integration.md)).
- **On demand**, via the `DemoQuestValidate` cheat command below.

Useful helpers for editor tooling: `CanAddPrerequisiteSafely(Quest, NewPrerequisite)` (check before letting a designer add a prerequisite), `GetDependencyTreeString(Quest)` (human-readable tree for debugging), `GetDependentQuests(Quest, AllQuests)` (reverse lookup — what breaks if this quest is removed).

## Loading quests

`UDemoQuestValidationLibrary::LoadAllQuests()` scans the whole Asset Registry for every `UDemoQuestData` asset. `FindQuestByID(QuestID)` uses it internally and is the standard way to resolve a quest from a save file or a console command:

```cpp
UDemoQuestData* Quest = UDemoQuestValidationLibrary::FindQuestByID(TEXT("Quest_LostSword"));
```

No `AssetManager` configuration is required for this lookup to work at runtime. The `AssetManager` "Quest" primary asset type entry in `DefaultGame.ini` still matters for **packaged builds** — it tells the cooker which quest assets to include — so keep it pointing at your actual quest folders even though it no longer affects in-editor/PIE lookups.

## Cheat commands

`UDemoQuestCheatManagerExtension` registers automatically with every `CheatManager` (non-shipping builds). It lives in the optional **`QuestSystemDemoDebug`** module (alongside the debug overlay), so the core `QuestSystemDemo` module stays free of any dev-only code. Use the commands from the in-game console. Each command that touches player state forwards itself to the server automatically if run on a client.

| Command | Usage | Purpose |
|---|---|---|
| `DemoQuestValidate` | `DemoQuestValidate` | Full dependency validation + diagnostics dump. |
| `DemoQuestShowDependencies` | `DemoQuestShowDependencies <QuestID>` | Print a quest's prerequisite tree. |
| `DemoQuestShowDependents` | `DemoQuestShowDependents <QuestID>` | Print quests that depend on this one. |
| `DemoQuestShowChain` | `DemoQuestShowChain <QuestID>` | Print the full (recursive) prerequisite chain. |
| `DemoQuestCanAddPrereq` | `DemoQuestCanAddPrereq <QuestID> <NewPrereqID>` | Check whether adding a prerequisite would create a cycle. |
| `DemoQuestStats` | `DemoQuestStats` | Aggregate stats: quests with prerequisites, average chain length, etc. |
| `DemoQuestStart` | `DemoQuestStart <QuestID>` | Force-start a quest on the calling player. |
| `DemoQuestComplete` | `DemoQuestComplete <QuestID>` | Force-complete a quest. |
| `DemoQuestFail` | `DemoQuestFail <QuestID>` | Force-fail a quest. |
| `DemoQuestListActive` | `DemoQuestListActive` | List the calling player's active quests. |
| `DemoQuestListAvailable` | `DemoQuestListAvailable` | List quests the player could start right now. |
| `DemoQuestListCompleted` | `DemoQuestListCompleted` | List the player's completed quests. |
| `DemoQuestResetAll` | `DemoQuestResetAll` | Clear all quest progress for the player. |
| `DemoQuestReset` | `DemoQuestReset <QuestID>` | Clear progress for one quest. |
| `DemoQuestStatus` | `DemoQuestStatus <QuestID>` | Full status report: objectives, timer, rewards, settings. |
| `DemoQuestListAll` | `DemoQuestListAll` | List every quest in the project. |
| `DemoQuestFind` | `DemoQuestFind <PartialName>` | Search quests by partial name/ID. |
| `DemoQuestOverlay` | `DemoQuestOverlay` | Toggle the translucent quest debug overlay (docked top-right, game keeps running; lists every quest with live state and per-card controls). |

`DemoQuestOverlay` is one command on the same extension; it opens the debug overlay from the **`QuestSystemDemoDebug`** module. You can also open it from code/Blueprint with `UDemoQuestDebugLibrary::ToggleQuestDebugOverlay(PlayerController)` — bind it to any key.

The overlay is a full inspector, not just a list: each card shows the quest's objectives, its own timer if it has one, sharing mode, a **"Routes to" badge** (which manager actually owns this quest's progress — `PlayerState`, `GameState (Party)`, or both — the single most common source of "quest not available/not progressing" confusion in this plugin, made visible instead of requiring you to know the rule), a **party roster/contributions line** for Shared and Individual quests (per-player contribution amounts, or per-player `[x]`/`[ ]` completion), **rewards**, and per-card controls (start / complete / fail / abandon / reset, objective +1 / fill). Each objective with its own `TimeLimit` shows its own live countdown too, entirely independent of the quest's — both count down for real, once per second, not just whenever something else happens to refresh the panel. Separately, whenever a quest or objective has a `TimeLimit`/`TimerWarningThreshold` configured (`> 0`), its card also shows the *static* **Limit**/**Warn** values themselves, in every state — not just while the live countdown is running. A **Locked** quest's state pill reads `LOCKED (N)` and its tooltip names the specific unmet prerequisites, instead of a generic "locked" — see [Prerequisites and validation](03-Authoring-Quests.md#prerequisites-and-validation) for the cross-sharing-mode routing rule behind that check. **Validate** (runs the dependency validator and shows PASSED/FAILED in a readout row right below) and **Reset All** (clears progress on every quest that has any) sit top-right, in the same row as the **Quests**/**Events** tabs, so both stay reachable no matter which tab is open. The Quests tab's own filter row adds a **SEARCH** box next to the **STATE**/**MODE** pills — like them, it only narrows the visible card list without affecting their pill counts. The box itself is deliberately narrow (about a quarter to a third of the panel's width), with a small **x** button to clear it and, to its right, seven checkboxes controlling exactly which fields the typed text is matched against (case-insensitive, per quest or any of its objectives): quest **Name**, **ID**, **Giver** (`QuestGiverDisplayName`), and **Description**, plus objective **Name**, **ID** (`ObjectiveIdentifier`), and **Description**. Name/ID/Giver/objective Name/objective ID are checked by default; the two free-form Description checkboxes start unchecked, since prose text tends to produce noisier incidental matches than an identifying name or ID. The game is **not** paused while the panel is open — you keep controlling your pawn, and the panel refreshes itself several times a second so quest/objective progress shows up live. Close it with the top-right button or the same key that opened it.

**Live event feed.** An **Events** tab (next to **Quests**, right under the header) swaps the whole panel body for a full-height, scrolling, newest-first feed from the local player's own manager(s) — no quest filters or the card list competing for space while it's open. Three kinds of rows: every `OnQuestEventNotified` (a raw `NotifyQuestEvent`/`NotifyPartyQuestEvent` call, whether or not it matched an active objective — unmatched ones highlighted in orange, the "why didn't my quest progress" case), every `OnQuestStateChanged` (a quest's overall state just transitioned), and every `OnObjectiveUpdated` (an objective's progress counter just changed). Its own **KIND** filter row (Event / Quest / Objective / All, with live per-kind counts) narrows the feed to one row type — useful once a Shared quest with several participants starts producing a steady stream of `OBJECTIVE` rows that would otherwise bury the rarer, more diagnostic ones. This only reflects the *local* player's managers (delegate broadcasts aren't replicated), so use it from the host/listen server or in standalone/PIE — see [04 — Event-Driven Progress](04-Event-Driven-Progress.md#listening-for-progress) for the underlying delegates.

## Where to go next

- Extending the plugin in C++ (custom rewards, custom components): [10 — C++ Integration](10-CPP-Integration.md)

<!-- doc-footer:start -->
---
*Generated 2026-08-04 19:47 UTC from `Docs/Full/` - do not edit this page directly.*
<!-- doc-footer:end -->
