# 03 — Interaction and focus

*🇬🇧 English | [🇺🇦 Українська](../uk/03-Interaction-and-Focus.md)*

## Source

> **In plain words.** The source is "a hand with a laser pointer". The pointer constantly shows what is in front of you (focus), and the hand picks the item up when you press a key. In a desktop game one such hand is tied to the camera; in VR there is one per controller.

`UInteractorComponent` is a scene component on a pawn. Attach it to a camera or a motion controller. With `bUsePawnViewpoint=true` it updates its own position from the current camera. With `false` its transform is determined by the parent component or by project code.

| Parameter | Purpose |
|---|---|
| `TraceDistance` | Ray length, 300 cm by default |
| `TraceRadius` | 0 — line trace; above 0 — sphere sweep |
| `FocusUpdateHz` | Focus search rate for this hand; 30 Hz by default. 20 Hz reduces the number of traces and CPU load |
| `PoseReplicationHz` | Rate of the shared pose packet of all active hands of the pawn; 30 Hz by default. 20 Hz reduces network traffic but increases the interval between updates on the server. The value of the first active interactor is used |
| `bNearInteraction` / `NearRadius` | Search for an item near the palm through an overlap, with a visibility check; convenient for VR |
| `TraceChannel` | Blocking channel, Visibility by default |
| `Filter.RequiredTags` | All required Gameplay Tags, with hierarchical matching |
| `Filter.BlockedTags` | Any of these Gameplay Tags forbids the target |
| `Filter.RequiredActorTags` | All exact names from Actor.Tags |
| `GrabOffset` | Offset of a grabbed physical item from the source; usually zero for VR |

A target has `IInteractableInterface` for an ordinary action or `UInteractionGrabTargetComponent` for grabbing. The ready-made `UInteractionManipulationComponent` implements the grab contract; it can be added to your own actor without inheriting from `AInteractionPhysicsActor`. If you need to describe the holding behavior yourself without C++, use the Blueprint subclass `UInteractionBlueprintGrabTargetComponent` ([11](11-Blueprint-Integration.md#custom-grab-target-in-blueprint)). If there is no interface, the filter reads the component's `TargetTags`. The local pose is updated every frame, independently of the focus and network rates.

A filtered or blocked target remains an obstacle for items behind it. With `TraceRadius > 0` the component calls `SweepSingleByChannel` with a sphere; the contact point is additionally checked with `LineTraceSingleByChannel` so that an item around a corner is not chosen. For such a search the target must block `TraceChannel`.

## Target selection order

`RefreshFocus` first looks for a grab target, and if there is none, a target for an ordinary action. An actor is suitable if:

1. it implements `IInteractableInterface` or has a `UInteractionGrabTargetComponent`;
2. its tags pass `Filter`: the interface tags (`GetInteractionTags`) or, without an interface, the component's `TargetTags`, as well as `Actor.Tags`;
3. `CanInteract` returns `true`, if there is an interface;
4. for grabbing — the target component allows the grip (`CanBeginGrip`): there is a free slot for a hand, a valid profile and an available grip point.

Near search (`bNearInteraction`) finds all objects in a sphere of `NearRadius` around the source, discards unsuitable ones, takes the nearest collision point for the rest and checks visibility to it. The nearest candidate wins. Without near search the first blocking object of the ray wins.

An actor with only a target component, without an interface, gets focus only when it can be picked up right now: it has no ordinary action. So, for example, another player's busy item does not show a prompt.

The server repeats the same search from its copy of the hand pose, so a target the client saw in focus may not be confirmed on the server. Then `OnActionResult` reports the rejection reason ([13](13-Network-Model.md#rejection-reasons)).

## Tag filtering

> **In plain words.** Tags are stickers on items, and the filter is a rule on the hand: "take only items with the *Item* sticker and never with the *Locked* sticker". So one hand can take only quest items and another can take anything.

The filter belongs to the **source** (`Interactor.Filter`), and the tags belong to the **target**. `GetInteractionTags()` returns the target's Gameplay Tags; the ready-made `AInteractionPhysicsActor` returns its `InteractionTags`. Separately, every Unreal actor has an ordinary `Tags` array of exact `FName` names. To select a target, all `RequiredTags`, none of the `BlockedTags` and all `RequiredActorTags` are needed. Empty lists restrict nothing.

For example, a hand should take only quest items. On the hand set `RequiredTags = {Interaction.Item}`, `BlockedTags = {Interaction.Locked}` and `RequiredActorTags = {QuestPickup}`. On a spear set `InteractionTags = {Interaction.Item}` and add `QuestPickup` to **Actor → Tags**. The spear passes the filter. If you add `Interaction.Locked` to it, it stops being selected; if you remove `QuestPickup`, the result is the same. The built-in tags `Interaction.Item` and `Interaction.Locked` are declared in `InteractionGameplayTags.h`.

The equivalent setup in C++ after creating the components:

```cpp
Interactor->Filter.RequiredTags.AddTag(InteractionGameplayTags::Item);
Interactor->Filter.BlockedTags.AddTag(InteractionGameplayTags::Locked);
Interactor->Filter.RequiredActorTags.Add(TEXT("QuestPickup"));

Spear->InteractionTags.AddTag(InteractionGameplayTags::Item);
Spear->Tags.Add(TEXT("QuestPickup"));
```

Gameplay Tags are hierarchical: a required `Interaction.Item` also matches a child tag such as `Interaction.Item.Spear`, if you registered it in the project. `Actor.Tags` are plain names: `QuestPickup` and `QuestPickup.Special` are different. The filter is checked both locally for focus and again on the server before the action. A rejected item still blocks the items behind it if its collision blocks `TraceChannel`.

## Target contract

| Method | Where it runs | Responsibility |
|---|---|---|
| `CanInteract` | Local query and again on the server | Allow the action; implement without side effects |
| `GetInteractionTags` | Locally and on the server | Return semantic tags |
| `Interaction` | Server | Open a door, press a button or another gameplay action |
| `OnFocusStart` / `OnFocusEnd` | Local observer | Show or remove visual feedback |
| `OnGrab(Interactor)` | Server, when the first hand is accepted | Tell gameplay about the pickup; the target component drives the motion |
| `OnRelease(Interactor, Reason, LinearVelocity, AngularVelocity)` | Server, when the last hand lets go | Tell gameplay about the release; `Reason = Released` means an intentional release, and the velocities are already clamped by the server |

The core calls `OnGrab` and `OnRelease` for any target with the interface, not only for ready-made actors. The interface events — `Interaction`, `OnFocusStart`, `OnFocusEnd`, `OnGrab`, `OnRelease` — can only be implemented, not called from Blueprint: this way a client cannot bypass the server check. Check admissibility in `CanInteract` and `CanBeginGrip`. For your own target component implement `BeginGrip`/`EndGrip`; for the ready-made manipulation use `OnGripStarted`, `OnGripEnded` and `OnUsed`, keeping a single motion executor.

`GetFocusedActor` returns one current target per source. The transition A → B sends the end of A before the start of B. `OnFocusChanged` is suitable for the HUD; `OnActionResult` reports the result of a server request together with the rejection reason (`EInteractionRejectReason`). Local focus does not mean the server has already allowed the action; how to explain a rejection to the player is in [lesson 3](14-Tutorials.md#lesson-3-telling-the-player-why-an-action-failed).

One `UInteractorComponent` drives one hand: while grabbing or waiting for the server's answer it clears focus from other items. `CanFocusNewTarget()` shows whether this hand can choose a new target right now. In VR the second, free hand has its own component and can choose independently. After release the first hand searches for targets again.

## Visual feedback

`AInteractionMasterItem` adds a proximity sphere, text markers and optional billboard textures. Set `NearbyIcon.Sprite` and `FocusIcon.Sprite` to replace the text. The billboard turns toward the camera by itself, including in stereo mode.

For highlighting, assign `FocusOverlayMaterial`. If needed, enable `bUseCustomDepthFocus`; a visible outline requires a post-process material in the project. Without it, custom depth draws nothing by itself. The previous overlay and stencil are restored when focus ends.

Focus of two hands is counted separately: ending the focus of one hand does not remove the other's highlight. The built-in presentation is shared by local players of one process. For independent split-screen UI use `OnFocusChanged` and your own widgets for each LocalPlayer.

The proximity sphere uses an overlap with the locally controlled pawn. Enable **Generate Overlap Events** on the character's capsule. The proximity marker appears only if at least one local hand is free; it does not give the right to interact through walls.
