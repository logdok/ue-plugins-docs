# 04 — Multiplayer and VR

*🇬🇧 English | [🇺🇦 Українська](../uk/04-Multiplayer-and-VR.md)*

## The server path

> **In plain words.** The client only asks: "give me this crate". The server, as the referee, checks whether the player is really nearby, whether they can see the crate, and whether the crate is already in someone else's hands — and only then allows it. So an honest player sees the prompt at once, while a cheater cannot reach an item through a wall.

RPCs live on the pawn component owned by the client. Items do not transfer network ownership for the sake of interaction. In a networked game the target must have `Replicates=true`.

The server checks the request interval, the target, the pose packet, whether the hand is busy and the claim ledger. Then it repeats the target search, the tag checks, `CanInteract` if there is an interface, and the target component's `CanBeginGrip`. The answer arrives in `OnActionResult` together with the rejection reason (`EInteractionRejectReason`): the table of reasons is in [13](13-Network-Model.md#rejection-reasons), and messages that make sense to a player are in [lesson 3](14-Tutorials.md#lesson-3-telling-the-player-why-an-action-failed).

Two hands of one pawn can hold one item if `MaximumHands=2`; two different pawns cannot. A release request must match the server's `HeldGrip.Target` and `HeldGrip.RequestId`. A late release of a previous grab does not free the new one. The reservation mechanism is described below.

## Network limits

The limits are set for the whole project in **Project Settings → Plugins → Interaction System** (`UInteractionSystemSettings`, file `Config/DefaultGame.ini`). A single `InteractorComponent` can override any value: turn on the checkbox next to the parameter in **Details** (`bOverrideMaxOriginDistance` and so on) and set your own.

| Parameter | Default | What it limits |
|---|---|---|
| `MaxOriginDistance` | 500 cm | Distance from the pawn to the reported camera or hand pose; take the camera boom length into account |
| `MinActionInterval` | 0.08 s | Minimum time between accepted requests of one source |
| `PoseTimeout` | 0.75 s | How long the server waits for any pose of a remote source before releasing its item |
| `MaxThrowSpeed` | 2500 cm/s | Largest linear throw speed |
| `MaxAngularSpeed` | 3600 degrees/s | Largest angular throw speed |

The effective values of a source are returned by `GetMaxOriginDistance`, `GetMinActionInterval`, `GetPoseTimeout`, `GetMaxThrowSpeed` and `GetMaxAngularSpeed`; they are also shown by the [debug panel](15-Debug-Tools.md). Change the limits consistently on the server and clients: the final decision always belongs to the server.

```ini
; Config/DefaultGame.ini of your project
[/Script/InteractionSystem.InteractionSystemSettings]
MaxOriginDistance=650
PoseTimeout=1.0
```

The server clamps excessive speeds. On loss of the controller, a pose timeout, deactivation or EndPlay of the source, the item is released without a throw. Physics, damage, destruction and spear embedding are computed on the server.

## Hand poses

The client regularly sends the server a packet of the poses of all the pawn's hands, and also attaches it to every request. Each hand has its **own** sample number, and the server validates each hand separately: distance from the pawn and visibility from the pawn's eyes. If a free hand ends up behind an obstacle, the server rejects only its sample, and the item in the other hand stays. The packet as a whole is rejected only when it is corrupted: someone else's hand, duplicates, more than eight entries.

> **In plain words.** You send the referee separate photos of each hand. If one photo is blurry, the referee distrusts only that one — the photo of the other hand still counts.

The full packet contents, validations and timeouts are in [13 — Network model](13-Network-Model.md#pose-packet).

Data about the client's direction remains client data. Geometric limits and the repeated trace constrain the permitted action but are not a complete anti-cheat system.

## The claim ledger `UInteractionClaimSubsystem`

> **In plain words.** The ledger is a cloakroom with tickets. Whoever got the ticket for an item first holds it; the second player gets the rejection `Claimed` until the ticket is returned.

`UInteractionClaimSubsystem` is an automatic subsystem of every game `UWorld`. It is the single server ledger of the right to an item: for each target it stores the owning pawn, one or two hands of that pawn and, if needed, the parent linked action. The ledger uses keys that do not extend the lifetime of actors, cleans up destroyed hands and targets, and never becomes the client's source of truth. Clients learn the result through `OnActionResult`, `HeldGrip` and the replicated state of the manipulation component.

For example, the standard spear has `MaximumHands=1`. When two hands send `RequestGrab` at the same time, the first valid claim reserves it; the second gets a rejection. For a rifle with `MaximumHands=2` the second hand of the same pawn can join. A hand of another player is rejected regardless of free space. A `RequestInteraction` from a hand that already holds the item calls its `Use`; a foreign source cannot perform an ordinary action on a reserved target.

| Method | Purpose |
|---|---|
| `TryClaim(Target, Interactor, MaximumHands)` | On the server adds a hand of the same pawn; the limit is one or two hands, another pawn always gets `false` |
| `FindClaimant(Target)` / `FindHands(Target)` | The first hand or all hands that hold the target |
| `FindOwningPawn(Target)` / `IsClaimed(Target)` | The owner and the fact of reservation, including a linked one without an active hand |
| `GetClaimedTargets()` / `FindLinkedParent(Target)` | All claimed targets; the parent item of a linked reservation, for example the bow for an arrow |
| `ReleaseClaim(Target, Interactor)` | Removes only this hand; the other hand stays the owner and becomes primary |
| `ReserveLinked(Target, Parent, Pawn)` | Reserves an arrow, piston or other child target for the same pawn and parent item; cycles are forbidden |
| `ReleaseLinked(Target, Parent)` / `ForgetTarget(Target)` | Ends a linked action and recursively clears dependent reservations |

The queries (`Find…`, `IsClaimed`, `GetClaimedTargets`) are available in Blueprint too, but only on the server. Change the ledger only through interactor requests: the component keeps the claim, `HeldGrip` and the callbacks consistent. On release the server calls the item component's `EndGrip` and removes only this hand's claim. The manipulation rebuilds the grip baseline for the remaining hand; the last hand triggers the release policy. A linked arrow reservation can persist without hands until it is taken off the string.

```cpp
if (UInteractionClaimSubsystem* Claims = GetWorld()->GetSubsystem<UInteractionClaimSubsystem>())
{
    UInteractorComponent* Hand = Claims->FindClaimant(Spear);
}
```

## Grab permission

`UInteractionManipulationComponent::bGrabEnabled` and `AInteractionPhysicsActor::bCanBeGrabbed` are replicated: it is enough to change them on the server, and clients stop showing the prompt. For manipulation use `Set Grab Enabled`: turning it off immediately and safely releases the current hands.

## Two VR hands

1. On a replicated pawn create two motion controller components.
2. Add an `InteractorComponent` under each controller. Turn off `Use Pawn Viewpoint`, keep `GrabOffset=(0,0,0)`.
3. Set the source's forward axis along local +X. If needed, add an intermediate scene component to correct the orientation.
4. Call `RequestGrab` from the grip/input action of the corresponding hand. For `RequestRelease` use the filtered linear and angular velocities of the tracked controller in world coordinates.
5. Replace the `E` texture with the controller symbol you need or use your own HUD.

`FocusUpdateHz` and `PoseReplicationHz` are set separately. Both default to 30 Hz: the first controls the focus search, the second the rate of the pose packet of both hands. 20 Hz reduces CPU, RPCs and network traffic but makes aiming and the remote hand less smooth; a held item is still solved every frame. The authoritative hand pose is replicated to other clients; the owner receives no correction of their tracked transform.

An item in the owner's hand moves without network delay in both modes: the owner solves `Direct` themself and predicts `Physics`. When the server body hits an obstacle further than `OwnerPredictionTolerance` (10 cm), the owner smoothly moves to the server pose and sees that a wall stopped the item ([08](08-VR-Manipulation.md#physics-mode-in-the-owners-hand)). The grab itself is not predicted: the item ends up in the hand after the server's answer.

VR locomotion, tracking origin, body replication and Enhanced Input bindings are determined by the project. Report pose validity through `SetTrackingValid`: a short loss holds the last pose for `TrackingLossGrace`, a longer one safely ends the grip. Before a teleport/recenter call `CancelGrip(Recentered)` so that old velocities do not become a throw. A real HMD is not part of the automated tests.

## A full VR example

The detailed scheme of profiles, socket/component/free anchors, two hands, the bow, the spatial panel, the wheel and the piston seating is described in [08 — VR manipulation](08-VR-Manipulation.md). The map `/InteractionSystem/Demo/VRLab/Maps/VRLab` works with OpenXR and has a desktop emulator of two hands, so it can be tested without a headset. For Quest 2 use the same pawn with `MotionControllerComponent`, `EHMDTrackingOrigin::LocalFloor` and Enhanced Input; before release check the Android build on your device.

## Verifying the network

The fastest way to see what is happening is the [debug panel](15-Debug-Tools.md): on a PIE client type `EnableCheats`, then `InteractionOverlay`. For latency and packet loss use `Net PktLag=150` and `Net PktLoss=5`.

The automated tests run a dedicated server and clients in one PIE process:

| Test | What it verifies |
|---|---|
| `Network.TwoClientsGrabMoveReleaseAndEmbed` | Simultaneous competition for an item, replication of the owner, movement of the remote hand, release, spear embedding, movement together with a surface and regrab |
| `Network.ObstructedFreeHandKeepsOtherGrip` | A free hand hidden by an obstacle does not make the server drop the item in the other hand |
| `Network.ClientDeactivatedSourceKeepsPoses` | A locally deactivated first hand does not stop pose updates of the second |
| `Network.GrabPermissionReplicates` | A grab permission disabled on the server reaches the client |
| `Network.OwnerPredictsPhysicsGrip` | The owner sees a physical item in their hand immediately, even with packet latency |
| `Network.PushModelReplicatesManipulation` | With push model on, a second client sees the movement and release of someone else's item |
| `Demo.SpearMeshThrowAndRetrieval` | The physical cycle with the SpearRange mesh; skipped if the demo content is removed |

This does not replace verification on different machines, with high latency, packet loss and late join in your game.
