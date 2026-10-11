# 13 — Network model

*🇬🇧 English | [🇺🇦 Українська](../uk/13-Network-Model.md)*

This chapter explains how InteractionSystem works in a networked game: what the client decides, what the server validates, what data is transferred and why an item can be rejected or released. The setup basics are in [04 — Multiplayer and VR](04-Multiplayer-and-VR.md); here is the internal mechanics needed for diagnostics and your own extensions. Unfamiliar terms are explained in the [glossary](Glossary.md).

> **In plain words.** The client is a player who sees the world and asks permission. The server is the referee with the final version of the world. The player at once sees their own prompt and the motion of the item in their own hand, but who holds what is written down only by the referee.

## Who decides what

| What | Where it is decided | How other machines learn |
|---|---|---|
| Focus, highlighting, prompts | Locally, on the player's machine | Not replicated |
| Camera and hand poses | The client reports, the server validates each hand separately | The unreliable RPC `ServerUpdatePoses` and a copy of the packet in every request; to other clients — the replicated transform of the interactor |
| Grab, use, release | Server | Reliable RPCs, replicated `HeldGrip`, `ClientActionResult` with a reason |
| Right to an item | Server, `UInteractionClaimSubsystem` | The ledger itself is not replicated; the result is visible through `HeldGrip` and the item snapshot |
| Item motion, scale, attachment | Server; the owner solves motion in their own hand (`Direct`) or predicts it (`Physics`) | The replicated snapshot of `UInteractionManipulationComponent` |
| Physics, embedding, impacts, damage | Server | Snapshot, multicast for sound and effects |

RPCs live on the `UInteractorComponent` of a pawn owned by the client. Items do not need to transfer network ownership.

## The grab request path

1. **Client.** `RequestGrab` checks that the component is active, the pawn is locally controlled, tracking is reliable and the hand is free. The component collects a packet of the poses of all active hands of the pawn, searches for the target with the same trace as focus, remembers it as pending, increments `LocalRequestId` and sends a reliable `ServerRequestAction(Target, Grab, RequestId, Poses)`.
2. **Server.** Checks the conditions in the order of the [rejection reasons](#rejection-reasons) table and returns the first reason that fired. If all checks pass, `UInteractionClaimSubsystem::TryClaim` reserves the target for the hand, the server writes `HeldGrip = {Target, RequestId}` and calls the target's `BeginGrip`. If `BeginGrip` returns `false`, the reservation and `HeldGrip` are cancelled and the client receives `Unavailable`. A target with `IInteractableInterface` receives `OnGrab` when the first hand takes it.
3. **Answer.** To every request that reached a component active on the server, the server replies with a reliable `ClientActionResult(Target, Action, bAccepted, RequestId, Reason)`, and `HeldGrip` arrives through replication. If the component is inactive on the server, the request is ignored without an answer.
4. **Client.** A rejection clears the pending target. `OnRep_HeldGrip` clears it only for the same `RequestId`, so a late answer to an old request does not cancel a new one. The `OnActionResult` event receives the result and the reason.
5. **Item.** On the server `UInteractionManipulationComponent` calls `OnGripStarted`, rebuilds the grip baseline and immediately sends the snapshot.

## Rejection reasons

Every server answer contains an `EInteractionRejectReason`. The server checks the conditions in this order and returns the first one that was not met:

| Reason | When |
|---|---|
| `Cooldown` | Less than `MinActionInterval` has passed since the previous accepted request of this source |
| `InvalidTarget` | The target does not exist or, in a networked game, is not replicated |
| `InvalidPose` | The pose packet is malformed or the pose of the source sending the action failed validation (distance, visibility from the pawn's eyes, timeout) |
| `HandBusy` | The hand already holds an item or the request is stale |
| `Claimed` | The target is held or reserved by another pawn |
| `NotReachable` | The server's repeated trace did not reach the target, although it could have been taken |
| `Unavailable` | The target refused: grabbing is off, there is no room, there is no ordinary action, or `BeginGrip` returned `false` |

`None` means the request was accepted. The reason is visible in the `OnActionResult` event and in the debug panel ([15](15-Debug-Tools.md)). How to turn the reasons into messages that make sense to a player — [lesson 3](14-Tutorials.md#lesson-3-telling-the-player-why-an-action-failed).

## An ordinary action

`RequestInteraction` has two variants:

- the hand holds an item — the target becomes this item without a trace, the server calls the grab target's `Use`, and the manipulation calls the `OnUsed` event;
- the hand is free — the server repeats the trace for `Interact` and calls `Interaction` if the target implements `IInteractableInterface` and nobody reserves it.

## Release and throw

`RequestRelease(Linear, Angular)` first discards infinite values. The client sends `ServerRelease` with the same `RequestId` it received the item with. The server performs the release only if the target and `RequestId` match `HeldGrip`. If the hand's pose failed validation or tracking is unreliable, an intentional release becomes `TrackingLost`, and there will be no throw.

On an intentional release (`Released`) the server clamps the velocities to `MaxThrowSpeed` and `MaxAngularSpeed` and passes them to `EndGrip`. The manipulation applies them only when the last hand releases the item, the profile has `bAllowThrow=true` and the body is simulated after the release. A target with the interface receives `OnRelease` with the reason and the clamped velocities. Your pawn computes the velocity in world coordinates; the demo VR pawn smooths the palm position difference between frames.

## Pose packet

A packet is an array of up to eight entries, one for each active `UInteractorComponent` of the pawn:

| Field | Meaning |
|---|---|
| `Source` | A reference to this pawn's interactor |
| `Sequence` | The sample number **of this source**; the server applies only those newer than the last accepted |
| `Position` | World position, quantized to 0.1 cm |
| `Orientation` | World rotation |
| `bTracked` | Whether the device pose is reliable now |

**Who sends.** The periodic packet is sent by the first active interactor of the pawn on the client at its own `PoseReplicationHz`. The same packet is attached to every action and release request. Because each source numbers its own samples, the server does not care who sent the packet: if the first hand is deactivated on the client, the next one starts sending packets and the numbering does not break.

**Packet validation.** A packet is rejected as a whole only when it is malformed: the pawn has no controller, there are zero entries or more than eight, an entry refers to someone else's or a duplicated source, or the source sending the action is absent from the packet.

**Per-sample validation.** The remaining checks apply to each hand separately:

1. the source is active on the server and its `Sequence` is newer than the last accepted; otherwise the sample is skipped without changes;
2. for `bTracked=true`: coordinates are finite, the position is no farther than `MaxOriginDistance` from the pawn;
3. the straight line from the pawn's eyes (`GetActorEyesViewPoint`) to the position does not cross geometry that blocks the source's `TraceChannel`. The pawn and the items in its hands are ignored. For near grip (`bNearInteraction`) touching a grab target closer than `NearRadius` (at most 30 cm) is allowed.

An accepted sample moves the server copy of the hand. A rejected sample marks only this hand: it keeps the last accepted pose, its actions are rejected with `InvalidPose`, and a release does not throw the item. The other hand works on independently.

> **In plain words.** If the left hand hid behind a door, the referee distrusts only the left hand. The rifle in the right hand stays in the right hand.

**What other players see.** The relative transform of the interactor is replicated to everyone except the owner (`COND_SkipOwner`): the owner receives no corrections of their own hand, and others see the server pose.

## Timeouts, tracking and forced release

The server releases an item without a throw in these cases:

| Condition | Reason |
|---|---|
| the pawn lost its controller | `Cancelled` |
| no sample at all arrives from a remote source for longer than `PoseTimeout` | `TrackingLost` |
| `bTracked=false` lasts longer than `TrackingLossGrace` | `TrackingLost` |
| `CancelGrip(Reason)` | `Reason`; `Released` is replaced with `Cancelled` |
| `Deactivate` or `EndPlay` of the interactor | `Cancelled` |
| the target became invalid: it fails `ValidateSetup`, the grip point disappeared, grabbing was turned off through `SetGrabEnabled`, the component is inactive | `Cancelled` |
| in `Physics` mode the body lagged behind the desired pose by more than `BreakDistance` over its closest approach | `Cancelled` |

The timeout counts silence: rejected samples also confirm that the hand is still in contact. `Recentered` and `TargetInvalid` are meant for your code: call `CancelGrip(Recentered)` before a teleport or recenter so that old velocities do not become a throw.

## Request identity

Every `RequestGrab` gets a new `LocalRequestId`. The server accepts only numbers newer than the last accepted for this hand and stores the target together with the number in `HeldGrip` atomically. A release must match both values. Therefore:

- a late release of the previous item does not release the new one;
- repeated delivery of an old request does not grab the item a second time;
- waiting for an answer (`HasRequestedGrip()`) blocks new focus and a new `RequestGrab` of this hand.

## Item state on clients

`UInteractionManipulationComponent` replicates one `State` structure: settings, hands and grip points, the number of hands, the grip baseline, the pose with scale, the lag of the physical body, the simulation flag, attachment, the external driver and the handle travel. The `Revision` field changes when the grip topology, attachment or physics must be applied again.

- **Compact pose.** The position is quantized to 0.01 cm, the scale to 0.01 (both `FVector_NetQuantize100`), and the rotation is sent as a compressed `FRotator`. Changes smaller than 0.01 cm, 0.01° or 0.001 of scale are not sent.
- **Push model.** The server marks `State` as changed only when the snapshot really changed. If push model is on (`net.IsPushModelEnabled=1` and a build with `WITH_PUSH_MODEL`, see [Performance](Performance.md#practical-tips)), an unchanged snapshot is not even compared during replication. Without push model the behavior is the same, only the server compares the snapshot every time.
- **Clients do not simulate the item's physics:** they smoothly move the pose toward the server's with `RemoteInterpolationSpeed`.
- **The owner** solves the pose from their own hands: for `Direct` always, for `Physics` while the server body does not lag by more than half of `OwnerPredictionTolerance`; beyond that they smoothly move to the server pose ([08](08-VR-Manipulation.md#physics-mode-in-the-owners-hand)).
- An attached item on a client follows its parent component. If the reference to the parent is not replicated, the client still receives the world pose from the snapshot.
- A player who joined later receives the full snapshot and `HeldGrip`, so they see the current owners and poses.
- Important changes call `FlushNetDormancy` and `ForceNetUpdate`. Do not make moving items dormant.

## Network roles

| Mode | Notes |
|---|---|
| Dedicated server | All checks and timeouts as described above |
| Listen server | The host's pawn is a local authority: actions run without RPCs, `OnActionResult` fires locally, there is no pose timeout for the host |
| Single-player | Like the host of a listen server; pose, distance and action interval checks are also performed |
| AI pawn on the server | `IsLocallyControlled()` returns `true`: focus is searched at `FocusUpdateHz`, actions run directly, `OnActionResult` fires on the server. If the AI does not need focus, reduce `FocusUpdateHz`; deactivating the component releases the held item |

## Actor requirements

- The pawn has `Replicates=true`; `UInteractorComponent` replicates itself.
- A target in a networked game has `Replicates=true`, and its collision blocks `TraceChannel`.
- A manipulation target is `Movable`; it is either not replicated or is a `UInteractionMeshComponent`.
- Do not enable `ReplicateMovement` on actors with `UInteractionManipulationComponent`: the component turns it off itself so that motion has one "writer".
- Components that items are attached to must be available for replication.
- `bGrabEnabled` and `bCanBeGrabbed` are replicated: change them on the server.

## What the server checks and what it does not

The server checks pose limits, visibility from the pawn's eyes, the interval between actions, the repeated trace, tags and `CanInteract`, the claim ledger, request and sample numbers, finiteness and speed limits. The server configuration has priority over the client's.

The server cannot verify exactly where the player aimed the hand within the permitted zone, and it does not check the throw direction. This is a limitation of any model with client-side tracking: geometric limits narrow a cheater's options but are not a complete anti-cheat system.

## Automated network tests

The `InteractionSystem.Network.*` tests run a dedicated server and clients in one PIE process: competition for an item and the full grab cycle, a free hand hidden by an obstacle, a locally deactivated first hand, replication of the grab permission and prediction of a physical item under packet latency. The list is in [04](04-Multiplayer-and-VR.md#verifying-the-network). Check separate machines, packet loss and late join manually, for example with the console commands `Net PktLag=100` and `Net PktLoss=5` in PIE, with the [debug panel](15-Debug-Tools.md).
