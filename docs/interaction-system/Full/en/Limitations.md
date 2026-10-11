# Limitations and known issues

*🇬🇧 English | [🇺🇦 Українська](../uk/Limitations.md)*

This page describes behavior of InteractionSystem 1.0 worth knowing before integration: deliberate limits of the model and scenarios that were not verified automatically. Each limit is an architectural decision with a reason; next to it is what to do if it gets in the way of your game.

## Network and VR

### There is no grab prediction

**Limit.** `RequestGrab` waits for the server's decision: the item appears in the hand after a server round trip. The same applies to `RequestInteraction`. After confirmation the item follows the owner's hand without delay: the owner solves `Direct` themself and predicts `Physics`.

**Why.** Who holds what is decided only by the server; a predicted grab would have to be rolled back when two players grab an item at once.

### Physics item prediction has a tolerance

**Limit.** The owner sees a physical item in their hand while the server body lags by no more than half of `OwnerPredictionTolerance` (by default 5 of 10 cm). If the body hits an obstacle, the owner smoothly moves to the server pose, so with high network latency the stop is seen with a delay. Other players always see the server pose, smoothed by `RemoteInterpolationSpeed`.

### Two hands — of one player only

**Limit.** An item can be held by at most two hands (`MaximumHands` 1 or 2) of one pawn. Joint holding by several players is not supported.

### Network targets must replicate

**Limit.** In a networked game the server rejects an action with an actor that has `Replicates=false` (reason `InvalidTarget`). A component that an item is attached to (`AttachToSurface`, a docking support) must be available for replication, otherwise the client will not attach the item but will only receive its pose from the snapshot.

### A hidden hand does not act until it can be seen

**Limit.** A hand whose pose the server rejected (far from the pawn or behind an obstacle from its eyes) keeps its last accepted pose. Its requests are rejected with `InvalidPose`, and a release does not throw the item. Other hands work independently. If no samples at all arrive from a source for longer than `PoseTimeout`, the server releases its item.

## Focus and targets

### One grab target per actor

**Limit.** The interactor finds the target through `FindComponentByClass<UInteractionGrabTargetComponent>()`, that is it takes the first such component of the actor, and the ledger keeps its accounting per actor. Do not add two manipulation components to one actor.

**What to do.** For one movable part of an actor use `TargetComponentName` ([08](08-VR-Manipulation.md#a-movable-part-inside-an-actor)). For several independent parts (two drawers of one cabinet) make each a separate actor attached to the cabinet.

### The first blocking object determines the obstacle

**Limit.** The ray and the sweep stop at the first object that blocks `TraceChannel`, even if it is not a target. Thin decorative meshes, triggers with Block on Visibility and invisible collision hide the items behind them. Set up the collision response or a separate trace channel.

## Manipulation

### The manipulation component turns off the whole actor's `ReplicateMovement`

**Limit.** On the server `BeginPlay` calls `SetReplicateMovement(false)` for the owner: motion is carried only by the manipulation snapshot. Do not turn `ReplicateMovement` on again. You can move the root from gameplay code: the component notices the change and passes it to clients; `SyncTransform()` does it immediately.

### Constrained handles

**Limit.** `Constraint=Rotary` and `Linear` work only with `Movement=Direct`, one hand and no scaling. A wheel's angle is computed around the target's pivot, so the mesh pivot must be at the center of rotation. A jump of more than 90° per frame does not count as a turn.

### Scaling

**Limit.** Scale is only uniform and positive, within `MinimumScale`–`MaximumScale` along the largest axis, only for `Direct`, two hands and no constraint. The target's parent component must be uniformly scaled.

### A Blueprint grab target does not move the item by itself

**Limit.** `UInteractionBlueprintGrabTargetComponent` only answers the server's questions and calls your events. The motion, attachment and physics are performed by your Blueprint itself, on the server, with the result replicated (for example the owner's **Replicate Movement** for attachment to a hand).

## Debug tools

**Limit.** The commands and panel of `InteractionSystemDebug` are unavailable in Shipping builds. On a PIE client they work after `EnableCheats`; the server commands (`InteractionClaims`, `InteractionResetAll`) are forwarded by the client through `ServerExec`, so the server must also allow cheat commands (in PIE it does).

## What was not verified automatically

- Building for Windows (MSVC) on the development machine: the name hiding check prevents typical errors but does not replace a build.
- A real Quest 2 and comfort in an HMD.
- Separate network machines, packet loss and late join. Packet latency is imitated by the automated tests in one PIE process.

Before releasing your game verify these scenarios yourself.
