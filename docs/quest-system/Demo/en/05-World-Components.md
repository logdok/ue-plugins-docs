# 05 — World Components

*🇬🇧 English | [🇺🇦 Українська](../uk/05-World-Components.md)*

Everything a level designer places in the world is a small, composable `UActorComponent` (or a tiny ready-made `AActor` wrapping one). Attach what you need to any actor — there is no required base class. Every class on this page lives in the `QuestSystemDemoWorld` module (a separate module from Core `QuestSystemDemo`, since none of it is Core quest logic — see [10 — C++ Integration](10-CPP-Integration.md) for the module list); include `DemoQuestSystemWorldTypes.h` for all of it at once, or individual headers.

## UDemoQuestGiverComponent

Offers quests to players.

| Member | Purpose |
|---|---|
| `AvailableQuests` (`TArray<UDemoQuestData*>`) | Quests this actor can offer. |
| `GiveQuest(Quest, Participants)` | Starts the quest. Routes automatically by `QuestSharingMode`: Personal goes to `Participants[0]`; Shared/Individual start a party quest for the whole array. |
| `CanGiveQuest(Quest, Player)` / `HasAvailableQuestsForPlayer(Player)` | Queries for UI/markers. |
| `GetAvailableQuests(Player)` | `BlueprintNativeEvent` — override to add custom filtering beyond prerequisites. |
| `ShouldOfferQuestToPlayer(Quest, Player)` | `BlueprintNativeEvent`, default `true` — override for level/reputation/flag gates. |
| `OnQuestAcceptedByPlayer(Quest, Player)` | `BlueprintNativeEvent` — override for dialogue/VFX/SFX. |
| `OnQuestAcceptedDelegate` | Multicast delegate for external listeners (e.g. a quest marker — see `UDemoQuestMarkerComponentBase` below). |

```cpp
UDemoQuestGiverComponent* Giver = CreateDefaultSubobject<UDemoQuestGiverComponent>(TEXT("QuestGiver"));
```

## UDemoQuestReceiverComponent

Accepts quest turn-ins. Mirrors the giver:

| Member | Purpose |
|---|---|
| `AcceptedQuests` (`TArray<UDemoQuestData*>`) | Quests that can be turned in here. |
| `TurnInQuestFromPlayer(Player, Quest)` | Completes the quest and triggers rewards. |
| `CanTurnInQuest(Quest, Player)` / `HasTurnInQuestsForPlayer(Player)` | Queries for UI/markers. |
| `GetTurnInQuests(Player)` | `BlueprintNativeEvent` — the mirror of the giver's `GetAvailableQuests`; override to add custom filtering beyond completion status. |
| `ShouldAcceptQuestFromPlayer(Quest, Player)` | `BlueprintNativeEvent`, default `true` — override for custom conditions. |
| `OnQuestTurnedInByPlayer(Quest, Player)` | `BlueprintNativeEvent` — override for celebration effects. |
| `OnQuestTurnedInDelegate` | Multicast delegate for external listeners. |

## Quest markers

Drives a visual indicator ("!" / "?") above a giver/receiver/target. This is a small class hierarchy, not one component with a mode switch:

- **`UDemoQuestMarkerComponentBase`** (abstract) — everything shared. It automatically finds a sibling `UDemoQuestGiverComponent` / `UDemoQuestReceiverComponent` / `UDemoQuestTargetComponent` on the same actor, tracks quest state for the local player, and exposes `GetMarkerState(Player)` / `GetRelevantQuest(Player)` (the specific quest driving the state — the one ready to turn in, or available — for tooltips/widgets that want to name it, not just show the state) / `GetLinkedGiverComponent()` / `GetLinkedReceiverComponent()` / `GetLinkedTargetComponent()`, plus an `OnMarkerStateChanged(NewState, OldState, RelevantQuest)` Blueprint event that fires on every state change, on every subclass. Enable `bUseEventDrivenUpdates` (default `true`, recommended) so it updates only when something actually changes rather than polling on a timer — it subscribes to **both** the player's personal manager and the party/GameState manager, so Shared and Individual quests refresh the marker exactly as promptly as Personal ones (see [routing quests to the right manager](06-Multiplayer.md)). For a world target (see above) the marker appears only while the player has an active objective the target would advance, and hides again once that objective is done. `bTrackInProgressQuests` (default off) shows an `InProgress` state for a quest associated with *this* actor — offered by its Giver or accepted by its Receiver, whichever is present — that the player has active but hasn't completed; works the same on a giver-only actor like `ADemoQuestBoard` as on a receiver-only or combo one. It also defaults to ignoring the parent's scale (`bAbsoluteScale`, inherited from `USceneComponent`, defaulted on in the base constructor) — since a marker is normally attached to the prop's own mesh, a bigger/smaller prop instance won't balloon or shrink the marker visual with it. Untick the component's own **Absolute Scale** (Transform category in the Details panel) if you ever want scale-following instead.
- **`UDemoQuestStaticMeshMarkerComponent`** — the built-in mesh subclass: assign `MarkerMeshComponent` (a proper component picker — pick an existing `StaticMeshComponent` already on the actor, by name) plus `AvailableMaterial` / `TurnInMaterial` / (optionally) `InProgressMaterial`, and it swaps materials and visibility for you. Leave `MarkerMeshComponent` unset and it falls back to a child literally named `"MarkerMesh"`, then the first `StaticMeshComponent` found among this component's own children (works for a Blueprint-added child regardless of name), then — if `bAutoCreateMarkerVisual` — auto-creates one at `BeginPlay` (using `DefaultMarkerMesh`, the engine cone by default); set `bAutoCreateMarkerVisual = false` if you'd rather get a warning than a silent default. Whichever path resolves it, the mesh actually in use is exposed read-only as `MarkerMesh`.
- **`UDemoQuestWidgetMarkerComponent`** — the built-in widget subclass: assign `WidgetClass` (a `TSubclassOf<UUserWidget>` — any Widget Blueprint, or a native `UUserWidget`) and the component shows/hides it for you, but has **zero opinion on its appearance** — unlike the mesh marker's per-state material slots, there's no built-in text/icon/color property here; all of that lives inside `WidgetClass` itself. Implement `IDemoQuestMarkerVisualInterface` (below) on the widget class to learn *why* it's showing (the new state, and the specific quest driving it) and branch on that however you like — swap a bound image, change text, play an animation. Its `WidgetComponent` resolves the same three-tier way as the mesh marker's `MarkerMeshComponent`: an explicit `MarkerWidgetComponent` picker reference first, then a child named `"MarkerWidget"`, then the first `WidgetComponent` among this component's children, then (if `bAutoCreateMarkerVisual`) auto-creating one — configurable via `WidgetSpace` / `DrawSize` / `Pivot` for the auto-created case (Screen space by default; switch to World for a billboarded/diegetic look).
- **`UDemoQuestBlueprintMarkerComponent`** — spawns and attaches a whole Actor (`MarkerActorClass`, typically a Blueprint — a floating arrow, an animated icon, particles, anything you can build in an Actor Blueprint) instead of a mesh or widget component. It toggles the spawned actor's visibility for you, and if that actor implements `IDemoQuestMarkerVisualInterface`, forwards state changes to it too — so it can do more than just show/hide (play an animation, swap a material, whatever). This is the one to reach for when you want a marker that's "a whole Actor" rather than a mesh or a widget, without leaving the plugin — and since `MarkerActorClass` is `TSubclassOf<AActor>`, that Blueprint can even add its own `Widget Component` directly in its Components panel (a billboarded UMG icon) with **zero C++ involved**: a Blueprint asset isn't compiled against the plugin's module, so it can use any plugin enabled in the project (UMG included) regardless of which C++ modules exist. The demo host's `AQuestDemoMarkerWidgetActor` + `UQuestDemoMarkerWidget` are a full working example of this pattern — a native C++ interface implementer driving a pure-C++ UMG widget (no WBP asset) that shows the marker state and quest name. Assign `AQuestDemoMarkerWidgetActor` to `MarkerActorClass` to see it live, or copy it as the reference for a Blueprint-only equivalent. For a marker that is *only* a widget, though, reach for `UDemoQuestWidgetMarkerComponent` above instead — it does the same job without spawning a separate actor.

**Driving a marker by hand.** The base also exposes the update machinery, on every subclass: `UpdateMarker(Player)` recalculates the state now, `SetMarkerState(State)` forces a state directly (bypassing detection — for cutscenes or scripted sequences), and `StartAutoUpdate()` / `StopAutoUpdate()` control the polling timer at runtime. The timer itself is configured by `UpdateInterval` (seconds, `0` = never poll) and `bAutoUpdate`; with `bUseEventDrivenUpdates` on, polling is only a safety net, so raising `UpdateInterval` — or switching `bAutoUpdate` off entirely and calling `UpdateMarker()` yourself — is the cheapest option for a level with many markers.

`IDemoQuestMarkerVisualInterface` (`Interfaces/DemoQuestMarkerVisualInterface.h`) is the one shared contract behind the last two bullets: a single `BlueprintNativeEvent`, `OnQuestMarkerStateChanged(NewState, RelevantQuest)`, implementable by whatever object actually visualizes the state — an Actor (for `UDemoQuestBlueprintMarkerComponent`) or a `UUserWidget` (for `UDemoQuestWidgetMarkerComponent`) alike.

There's no mode switch, so nothing stops you from adding **several marker components to the same actor** — e.g. a `UDemoQuestStaticMeshMarkerComponent` cone and a `UDemoQuestWidgetMarkerComponent` name-tag at once, each tracking the same quest state independently and drawing its own thing.

### Adding your own marker type

Subclass `UDemoQuestMarkerComponentBase` and override two protected virtuals: `SetupVisualization(Owner)` (called once from `BeginPlay`, after Giver/Receiver/Target siblings are cached — find or create whatever visual component you need) and `ApplyVisualization(State, RelevantQuest)` (called whenever the state changes — update it). The three built-in subclasses' own source are reference implementations of this pattern; if you just want "no built-in visual, only `OnMarkerStateChanged`", a one-off Blueprint subclass of `UDemoQuestMarkerComponentBase` needs neither override.

A native C++ subclass with a typed `UWidgetComponent`/`UUserWidget` property needs a `UMG` dependency — that's exactly what `UDemoQuestWidgetMarkerComponent` above already is, living in the `QuestSystemDemoWorld` module (which depends on `UMG`, unlike core `QuestSystemDemo`). Reach for a subclass of your own only when that built-in component genuinely doesn't fit — e.g. a marker driving more than a single mesh or widget component.

## UDemoQuestInteractorComponent — the player side

Attach to the player's pawn, bind one input action to `TryInteract()`, and you're done:

```cpp
QuestInteractor = CreateDefaultSubobject<UDemoQuestInteractorComponent>(TEXT("QuestInteractor"));

// in SetupPlayerInputComponent, any input system:
PlayerInputComponent->BindKey(EKeys::E, IE_Pressed, this, &AMyCharacter::Interact);

void AMyCharacter::Interact() { QuestInteractor->TryInteract(); }
```

`TryInteract()` is safe to call on a client (it forwards to the server via RPC). On the server it scans `InteractionRadius` centimeters around the pawn and, in priority order, acts on the closest actor that offers something:

1. A `UDemoQuestReceiverComponent` with a quest ready to turn in → turns it in.
2. A `UDemoQuestGiverComponent` with an available quest → accepts it (Personal quests go to the interacting player; Shared/Individual quests start for every currently connected player — see [06](06-Multiplayer.md)).
3. A `UDemoQuestTargetComponent` that answers to the Interact trigger → fires that component's quest event.

`OnQuestInteractionPerformed(TargetActor, Quest)` fires on the server after a successful interaction (`Quest` is null for a plain interaction target).

## UDemoQuestTargetComponent — the one smart world target

Drop this on **any** actor (a bottle, a pickup, an enemy, an NPC, a prop) and configure it in the Details panel. It is the single mechanism for "this actor is a quest target", organised around three questions:

**1. What does it report?** — the quest event, i.e. the standard `NotifyQuestEvent` wire:

| Member | Purpose |
|---|---|
| `SourceObjective` | **Optional.** Link the `UDemoQuestObjectiveData` this target serves and the three fields below are auto-filled from it and **locked** (greyed out) — no hand-typed string to mistype, and the identity can never drift from the objective asset. Editor-only convenience; runtime still uses the copied values. Leave empty to set the fields by hand. |
| `EventType` | Dropdown: Collect / Kill / Interact / Location / Custom (maps to the canonical `DemoQuestEvents::` IDs — pick the verb, not a magic string). |
| `CustomEventID` | Only when `EventType = Custom` (matches a Custom objective's `ExpectedEventID`). |
| `TargetID` | Matched against the objective's `ObjectiveIdentifier` — the one string that must line up. |
| `Count` | Progress reported per trigger (default `1`). |

**2. How is it triggered?** — `TriggerSources` is a bitmask; combine as needed, and a manual `Trigger(PlayerState)` always works on top:

| Flag | Fires when… |
|---|---|
| `Overlap` | a player walks into the target (a pawn-overlap sphere of `OverlapRadius` is spawned for you). |
| `Damage` | the owner takes standard `ApplyDamage` (shoot / hit it). No manual double-count guard needed — see `ConsumeMode`. |
| `Interact` | the player uses `UDemoQuestInteractorComponent` (the E key) near it. |
| `Destroyed` | the owner is destroyed (credited to its instigator, if any). |
| *(manual)* | your own combat/inventory code calls `Trigger(PlayerState)`. |

**3. What happens afterwards?** — the response:

| Member | Purpose |
|---|---|
| `ConsumeMode` | `Once` (fire a single time — built-in double-count guard), `Respawn` (+`RespawnTime`: hide, then reappear), or `Repeatable` (farmable). |
| `bHideOwnerOnConsumed` / `bDestroyOwnerOnConsumed` | Optional cleanup after a `Once` trigger. |
| `EffectToSpawn` | Optional actor spawned at the target for feedback (shards, VFX, SFX). |
| `OnQuestTargetTriggered(By)` | **BlueprintImplementableEvent on the target's own actor** — run custom reactions (hide a held item, flee, play a line) with no player-side wiring. |

Server-authoritative: every path is processed only where the owner has authority (the owner should replicate so hide/destroy reaches clients). The reported event still travels the normal `NotifyQuestEvent` route, so shared-credit matching is unchanged.

```
// Bottle you shoot:  EventType=Kill,  TargetID="Bottle",  Trigger=Damage,   Consume=Once + Destroy
// Letter you walk over: EventType=Collect, TargetID="Letter", Trigger=Overlap, Consume=Once + Destroy
// NPC you rob (press E): EventType=Collect, TargetID="Letter", Trigger=Interact, Consume=Once
//   → react in OnQuestTargetTriggered on the NPC
```

Because the trigger is independent of the reported event, letters lying on the ground (Overlap) and letters carried by NPCs (Interact) can report the same `Collect`/`"Letter"` event and feed a single objective.

## ADemoQuestLocationTrigger

A ready-made actor for `ReachLocation` objectives. Drop it in the level, set `LocationID`, and a box overlap reports `NotifyLocationReachedEvent` for whichever player enters it. `bDestroyAfterActivation` turns it into a one-shot trigger instead of a reusable checkpoint.

## ADemoQuestTargetActor

A batteries-included placeable actor: a visible mesh, a `UDemoQuestTargetComponent`, and a `UDemoQuestStaticMeshMarkerComponent` (auto-provisioning its own icon, see above) — ready to drop into a level and configure in the Details panel. Reach for this when the target is a dedicated prop you are placing purely for the quest (a pickup, a breakable, a touch-point). When the target is something that already exists in your game (an enemy, an NPC), add `UDemoQuestTargetComponent` (and, if you want a marker, any of the marker subclasses above) to that actor directly instead — the actor is only a convenience wrapper.

If your game already has its own combat or inventory systems, you can also skip world actors entirely and call `NotifyKillEvent` / `NotifyCollectEvent` (or `UDemoQuestTargetComponent::Trigger`) directly from that code.

## ADemoQuestBoard / ADemoQuestChest

Two more batteries-included placeable actors, for a giver-only or receiver-only prop: a visible mesh plus a `UDemoQuestGiverComponent` (`ADemoQuestBoard`, a notice board) or `UDemoQuestReceiverComponent` (`ADemoQuestChest`, a turn-in point), plus a `UDemoQuestStaticMeshMarkerComponent` — ready to drop into a level and configure in the Details panel. Content-free like the rest of `Actors/`: `PropMesh` ships with no default mesh, so assign one per-instance or in a Blueprint subclass; the marker auto-provisions its own icon if you don't assign one either (see [Quest markers](#quest-markers) above). Configure `AvailableQuests` / `AcceptedQuests` directly on the giver/receiver component.

## Composing your own NPC

None of the above requires a special base class — compose whichever components you need on any `AActor` or `ACharacter`. A quest-giving-and-receiving NPC is just a `Character` with `UDemoQuestGiverComponent` + `UDemoQuestReceiverComponent` + `UDemoQuestStaticMeshMarkerComponent` added on top — there's no dedicated ready-made class for this in the plugin (unlike `ADemoQuestBoard`/`ADemoQuestChest` above), since an NPC needs a skeletal mesh and animations, which are inherently game-specific content the plugin doesn't ship. Build it as a Blueprint (or a thin native `ACharacter` subclass, if you prefer C++) in your own project.

Look at `DemoMap` in the editor to see `ADemoQuestBoard`/`ADemoQuestChest` and a demo NPC Blueprint configured with real quests.

## Where to go next

- How giver/receiver route Personal vs. Shared/Individual quests under the hood: [06 — Multiplayer](06-Multiplayer.md)

<!-- doc-footer:start -->
---
*Generated 2026-08-05 14:13 UTC from `Docs/Full/` - do not edit this page directly.*
<!-- doc-footer:end -->
