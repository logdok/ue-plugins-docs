# 11 — Blueprint integration

*🇬🇧 English | [🇺🇦 Українська](../uk/11-Blueprint-Integration.md)*

This chapter shows the full path without C++: a character that picks up and throws items, an item, a door, a custom reaction to grabbing and two VR hands. Node names are given as the Blueprint editor shows them. The complete list of properties is in the [API reference](API-Reference.md).

## Before you start

- The plugin is enabled in **Edit → Plugins → InteractionSystem** and the editor has been restarted.
- For a networked game the pawn and all targets have **Replicates**. For a single-player game this is not required, but it does no harm.
- Target collision blocks the `Visibility` channel (or whichever `TraceChannel` you choose).

## A desktop character

### The component

1. Open the character Blueprint and in **Components → Add** add an `InteractorComponent`. Drag it onto the camera so that it follows the gaze.
2. Keep `Use Pawn Viewpoint` on: the component takes the controller's view point by itself.
3. If needed change `Trace Distance` (300 cm) and `Trace Radius` (0 — a thin ray; 5–10 — a wider sweep that makes it easier to hit small items).
4. `Grab Offset` determines where a taken item is held relative to the camera. The value `(70, 0, -15)` from InteractionLab holds the item in front of the player slightly below the screen center.

### Input with Enhanced Input

1. Create **Content Browser → Add → Input → Input Action** `IA_Use` and `IA_Throw` of type **Digital (bool)**.
2. Create an **Input → Input Mapping Context** `IMC_Interaction`: `E` → `IA_Use`, `F` → `IA_Throw`.
3. In the character add the event **Event Receive Controller Changed**. If `New Controller` is a `PlayerController` and the pawn `Is Locally Controlled`, call **Get Enhanced Input Local Player Subsystem → Add Mapping Context** with `IMC_Interaction`. This event also fires when the controller arrives on a client after `BeginPlay`.
4. Handle **IA_Use → Started**:
   - if `Get Held Actor` is valid, call `Request Interaction`: the held item receives `On Used`;
   - otherwise call `Refresh Focus`, take `Get Focused Actor`, do **Cast To InteractionPhysicsActor** and check `Can Be Grabbed`; if so, `Request Grab`, otherwise `Request Interaction`.
5. Handle **IA_Throw → Started**: if `Get Held Actor` is valid, call `Request Release` with `Linear Velocity = Get Forward Vector (camera) × 900` and `Angular Velocity = (0, 120, 0)`. Zero vectors simply release the item without a throw.

This is how `AInteractionDemoCharacter` from InteractionLab works; only the way input is delivered differs.

### HUD prompts

1. Select the `InteractorComponent` and in **Details → Events** click **+** next to `On Focus Changed`. The event fires only on the player's machine and passes `Previous Actor` and `New Actor`. Show a prompt when `New Actor` is valid and hide it when it is empty.
2. Add `On Action Result`: `Target Actor`, `Action`, `Accepted`, `Reason`. If `Accepted = false`, the server rejected the action, and `Reason` explains why: `Claimed` — the item is in another player's hands, `NotReachable` — out of reach, `InvalidPose` — the hand is too far or behind an obstacle, and so on. Connect **Switch on EInteractionRejectReason** and show a short message; a ready table of texts is in [lesson 3](14-Tutorials.md#lesson-3-telling-the-player-why-an-action-failed).
3. While the hand holds an item or waits for an answer, focus is empty: `Can Focus New Target` returns `false`. For a "Throw" prompt check `Get Held Actor`.

## An item that can be picked up

### Option 1: a ready-made class

1. **Content Browser → Add → Blueprint Class → All Classes** → `InteractionMasterItem` (with markers and highlighting) or `InteractionPhysicsActor` (without presentation).
2. Assign `Mesh Component` a static mesh with simple collision.
3. Create a Data Asset from `InteractionPhysicsDataAsset` (**Add → Miscellaneous → Data Asset**) and set `Mass In Kg`, `Start With Physics`, `Hit Sound`. Assign it in `Object Data`.
4. For an ordinary item the `Manipulation` settings are enough. For two hands, scaling or physical holding assign `Manipulation → Profile` (see [08](08-VR-Manipulation.md)).

Note: the ready-made physics actor plays `Hit Sound` on strong impacts and deals no damage until you turn on `Apply Impact Damage` in `Object Data` ([05](05-Physics.md#impacts-sound-and-damage)).

### Option 2: your own actor with components

1. Create a Blueprint Actor and enable **Replicates**.
2. The root component is a Static Mesh with **Mobility = Movable** and simple collision. Do not enable **Component Replicates** for it: the manipulation carries the motion. If the mesh must replicate (materials, visibility), use `InteractionMeshComponent` as the root.
3. Add an `InteractionManipulationComponent`. Without a profile it uses `Inline Settings`; for one hand set `Maximum Hands = 1`.
4. Set the tags for filters in the component's `Target Tags`.
5. For exact grip places add an `InteractionGripPointComponent` and choose `Grip Selection = Authored` or `Prefer Authored`.

An actor without `IInteractableInterface` does not receive `On Focus Start/End` and has no ordinary action, so focus on it appears only when it can be picked up right now. For highlighting or your own prompt implement the interface.

## A door or a button

1. Create a Blueprint Actor with **Replicates**, a mesh with collision and **Mobility = Movable**.
2. **Class Settings → Interfaces → Add** → `InteractableInterface`.
3. Add a variable `bOpen` with **Replication = RepNotify**. In the function `OnRep_bOpen` turn the leaf (a Timeline or `Set Relative Rotation`).
4. Implement **Event Interaction**: `Set bOpen = NOT bOpen`. The event runs on the server, so RepNotify carries the change to clients. On the server in Blueprint `OnRep` is called automatically after a `Set` with RepNotify.
5. Override `Can Interact`: return `true` for `Action = Interact` and `false` for `Grab`. The function also runs on the client for focus, so it must not change anything.
6. Optionally implement `On Focus Start` and `On Focus End` for local highlighting: they run only on the machine of the player whose hand aimed.

A simpler way is a child class of `InteractionPhysicsActor` with `Can Be Grabbed = false`: bind to `On Interacted` and change the replicated variable the same way. The main rule: everything other players see must be changed through replication, not directly in `Interaction`.

## Reacting to grab, use and release

On the item's `InteractionManipulationComponent` (**Details → Events**):

| Event | When | Typical use |
|---|---|---|
| `On Grip Started (Hand, First)` | A hand took the item; `First = true` for the first hand | Pickup sound, turning off the item's AI behavior |
| `On Grip Ended (Hand, Last, Reason, Linear Velocity, Angular Velocity)` | A hand released the item | A sound, starting a timer, returning to a place; `Reason = Released` means an intentional release |
| `On Used (Hand)` | The hand holding the item called `Request Interaction` | A shot, turning on a flashlight, pressing a button on a remote |
| `On Handle Changed (Value, Fraction)` | The travel of a rotary or linear handle changed | Open a door when `Fraction` reaches 1 |

All these events fire **only on the server**. Pass visual consequences to clients through replicated variables or multicast events.

An actor with `InteractableInterface` additionally receives the interface events `On Grab` (the first hand took it) and `On Release` (the last hand released it, with `Reason` and velocities) — also on the server.

A ready-made example is the wheel and door of VRLabBP: `BP_BunkerWheel` listens to `On Handle Changed`, and after unlocking `BP_BunkerDoor` calls `Cancel All Grips`, `Set External Driver` and every frame `Set External Pose`, so that the wheel moves together with the leaf ([09](09-VRLabBP.md)).

## Server-side manipulation functions

These functions of `InteractionManipulationComponent` are available in Blueprint and run only on the server (on a client the node does nothing):

| Function | Purpose |
|---|---|
| `Set Grab Enabled` | Forbid or allow grabbing; turning it off releases the current hands, clients see the new value |
| `Reset Manipulation` | Release the hands and return the item home ([08](08-VR-Manipulation.md#returning-to-the-home-pose)) |
| `Set Home Transform` / `Get Home Transform` | Change or read the home pose |
| `Attach To Surface` / `Is Attached` | Attach a free item to a component of another actor; regrabbing removes the attachment |
| `Set Free Simulation` | Remove the attachment and turn simulation on or off, optionally with an initial velocity |
| `Set External Driver` / `Set External Pose` | Hand the motion to another actor, like the wheel on a door |
| `Cancel All Grips` | Release all hands without a throw |
| `Sync Transform` | Immediately send clients a pose changed by your code |

The network limits of sources — distance, interval, throw speeds — are set in **Project Settings → Plugins → Interaction System** and overridden on a single `InteractorComponent` ([04](04-Multiplayer-and-VR.md#network-limits)).

## Custom grab target in Blueprint

If an item must behave in the hand differently from what the manipulation can do, create **Blueprint Class → All Classes → InteractionBlueprintGrabTargetComponent** and override its functions:

| Function | Where it runs | What to do |
|---|---|---|
| `Can Begin Grip` | Client (for the prompt) and server | Return `false` to forbid the grip; read only replicated state |
| `Begin Grip` | Server | Attach or move the owner; `false` cancels the grip (the client gets `Unavailable`) |
| `End Grip` | Server | Release the owner; `Reason = Released` is an intentional release |
| `Use` | Server | The action of the item in the hand; return `true` if handled |

The component itself checks `Grab Enabled`, free space (`Maximum Hands`) and that the second hand belongs to the same player. `Get Hands` and `Get Hand Count` are replicated; `Set Grab Enabled` and `Cancel All Grips` are called on the server. A full example is the lantern from [lesson 5](14-Tutorials.md#lesson-5-a-blueprint-lantern-a-custom-grab-target-without-c).

## Two VR hands

1. In the pawn add two `MotionControllerComponent`s (`Motion Source`: `Left` and `Right`) under a shared scene component at floor level.
2. Under each controller add an `InteractorComponent`: `Use Pawn Viewpoint = false`, `Hand = Left/Right`, `Near Interaction = true`, `Grab Offset = (0, 0, 0)`. If needed calibrate `Palm Offset`.
3. In **Event Tick** for each hand:
   - `Set Tracking Valid(Is Tracked)` of the controller;
   - remember `Get Grip Transform → Location` and compute the velocity as the difference from the previous frame divided by `Delta Seconds`. Smooth it (`VInterp To`) — this is the throw velocity.
4. Grip pressed → `Request Grab`; released → `Request Release(velocity, angular velocity)`.
5. Trigger → `Request Interaction`: using the item in the hand or an ordinary action with a free hand.
6. Before a teleport or recenter call `Cancel Grip` with `Reason = Recentered` for both hands.

A ready-made pawn with all of this and a desktop emulator is `InteractionVRPawn`; it can be taken as a parent class, like `BP_VRLabPawn` in VRLabBP.

## Common mistakes

| Symptom | Cause |
|---|---|
| There is focus, but the server rejects the action | Look at `Reason` in `On Action Result` or the `InteractionOverlay` panel ([15](15-Debug-Tools.md)): a target without **Replicates** (`InvalidTarget`), a hand farther than `Max Origin Distance` or behind an obstacle (`InvalidPose`), a target in someone else's hands (`Claimed`) |
| An item is not selected | Collision does not block `Visibility`; the actor has neither an interface nor a target component; `Filter` fired |
| A door opens only for the host | The leaf is turned directly in `Interaction` without replication |
| An item "twitches" for clients | `Replicate Movement` or replication of the root component is on next to the manipulation |
| A VR hand searches for items from the camera instead of the palm | The hand's component still has `Use Pawn Viewpoint = true` |
| After a teleport an item flies off | `Cancel Grip(Recentered)` was not called before moving |

More answers are in the [FAQ](FAQ.md) and the [limitations](Limitations.md).
