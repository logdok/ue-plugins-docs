# Release Notes

*🇬🇧 English | [🇺🇦 Українська](../uk/Release-Notes.md)*

A short summary of each release — what is new, what is fixed, and anything worth knowing
before you upgrade. Newest first.

---

## 1.0

The first public release. **Unreal Engine 5.7–5.8.**

### What's in it

**Quests are data, not code.** Author quests and objectives as Data Assets in the Content
Browser — `UDemoQuestData` and `UDemoQuestObjectiveData`. A designer can build and change quest content
without opening the project in an IDE. See [03 — Authoring Quests](03-Authoring-Quests.md).

**No setup to integrate.** `UDemoQuestWorldSubsystem` attaches a quest manager to every player and
to the game state by itself, so the plugin does not require a custom `PlayerState` or
`GameState` and does not ask you to rebuild your player class around it. See
[02 — Zero-Config Integration](02-Zero-Config-Integration.md).

**Progress is event-driven.** Your gameplay code reports what happened —
`NotifyQuestEvent` and its helpers — and the plugin decides which objectives that advances,
including sequential objectives and shared kill credit. See
[04 — Event-Driven Progress](04-Event-Driven-Progress.md).

**Multiplayer is built in, not bolted on.** Quest state replicates, party quests share progress
between members, and a single map can host single-player and multiplayer play at once. See
[06 — Multiplayer](06-Multiplayer.md).

**World components for the common cases.** Quest Giver, Quest Receiver, their marker
subclasses, Interactor, Quest Target and Location Trigger — plus ready-made actors when you
want to drop something into a level and move on. See
[05 — World Components](05-World-Components.md).

**Save and load on your terms.** `ExportState` / `ImportState` hand you the quest state as data
to embed in whatever save system you already have, rather than imposing one. See
[07 — Serialization](07-Serialization.md).

**Everything is available from Blueprint.** The full `UDemoQuestBlueprintLibrary` covers the same
surface C++ has. See [08 — Blueprint Library Reference](08-Blueprint-Library-Reference.md).

**Content problems are caught early.** Circular dependencies between quests are detected, and
console commands let you drive quest state while testing. See
[09 — Validation & Cheats](09-Validation-And-Cheats.md).

**Extension points where you need them.** Custom rewards, extended components, and sending
events from your own systems. See [10 — C++ Integration](10-CPP-Integration.md).

### Upgrading

Nothing to upgrade from — this is the first release. Start at the
[Quick Start](README.md#quick-start).

---

The plugin's own `CHANGELOG.md` is not shipped separately; this page is the record of what each
release contains.
