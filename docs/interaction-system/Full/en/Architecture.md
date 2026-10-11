# Architecture

*🇬🇧 English | [🇺🇦 Українська](../uk/Architecture.md)*

## Modules

| Module | Type | Responsibility | Dependencies |
|---|---|---|---|
| InteractionSystem | Runtime | Target contracts, filtering rules, the ray source, network requests, the claim ledger, project settings, the Blueprint grab target, grip math | Core, CoreUObject, Engine, GameplayTags, DeveloperSettings |
| InteractionSystemWorld | Runtime | Manipulation and profiles, grip points, bow, docking support, strikes, destruction, physics actors, master item, spear | Core, CoreUObject, Engine, GameplayTags, InteractionSystem; privately NetCore, Niagara |
| InteractionSystemDemo | Runtime | InteractionLab, SpearRange, VR Lab, Quest 2/OpenXR pawn, HUD, doors and example items | Core, CoreUObject, Engine, InputCore, InteractionSystem, InteractionSystemWorld, HeadMountedDisplay, XRBase, EnhancedInput, UMG, Slate, SlateCore |
| InteractionSystemDebug | Runtime | Console commands, the log of server answers and the debug panel; registers nothing in Shipping | Core, CoreUObject, Engine, InteractionSystem; privately InteractionSystemWorld, InputCore, UMG, Slate, SlateCore |
| InteractionSystemTests | Editor | Behavior and PIE network tests through the public API | Core, CoreUObject, Engine, GameplayTags, InputCore, UMG, UnrealEd, InteractionSystem, InteractionSystemWorld |

```text
InteractionSystem  <-  InteractionSystemWorld  <-  InteractionSystemDemo
        ^                        ^             <-  InteractionSystemDebug
        +---- InteractionSystemTests (public API only; Demo and Debug by name, without a dependency)
```

The core does not import World, UI libraries or editor modules. Rendering and the physical implementations of targets are outside the contract. Demo and Debug are leaf modules: nothing depends on them, so they can be removed ([06](06-Demo.md#excluding-from-the-packaged-game), [15](15-Debug-Tools.md#how-to-remove-the-module)); the tests reach them only by class names and skip their checks if the module is absent. The pure rules `FInteractionFilter`, `FInteractionRules` and `FInteractionGripMath` do not touch the world or RPCs; the UE scene/trace/network adapter is `UInteractorComponent`. The plugin enables the dependent plugins `EnhancedInput` and `XRBase`.

There is no separate use-case class for every call forward: protocol boundaries are placed where responsibility changes. Creating components in a pawn/Blueprint and configuring targets is the composition root of a specific game.

## Class roles

| Role | Class | Module |
|---|---|---|
| Action source: focus, requests, poses of one hand or camera | `UInteractorComponent` | Core |
| Contract for an ordinary action and local focus | `IInteractableInterface` | Core |
| Grab target contract | `UInteractionGrabTargetComponent`; the Blueprint variant `UInteractionBlueprintGrabTargetComponent` | Core |
| Project network limits | `UInteractionSystemSettings` | Core |
| Server ledger of the right to a target | `UInteractionClaimSubsystem` | Core |
| Rules and math without the world | `FInteractionFilter`, `FInteractionRules`, `FInteractionGripMath` | Core |
| Motion executor and the item's replicated snapshot | `UInteractionManipulationComponent`, `UInteractionManipulationProfile`, `UInteractionGripPointComponent` | World |
| Linked actions | `UInteractionBowComponent`, `UInteractionDockComponent` | World |
| Gameplay events without a damage decision | `UInteractionStrikeComponent` | World |
| Simple gameplay reactions | `UInteractionDestructibleComponent`, impact damage of `AInteractionPhysicsActor` | World |
| Ready-made actors | `AInteractionPhysicsActor`, `AInteractionMasterItem`, `AInteractionSpearActor` | World |
| Debugging | `UInteractionCheatManagerExtension`, `UInteractionDebugLibrary`, the answer log and the panel | Debug |

## Data flows

### Focus (local only)

```text
Tick interactor
  -> UpdateViewpoint (desktop) or the pose of the parent motion controller (VR)
  -> RefreshFocus at FocusUpdateHz
       -> TraceTarget(Grab); if no target is found -> TraceTarget(Interact)
  -> SetFocus: OnFocusEnd(old) -> OnFocusStart(new) -> OnFocusChanged
```

Focus is not replicated and is never proof of permission: the server repeats the checks itself.

### An action

```text
Input -> RequestGrab / RequestInteraction / RequestRelease
  -> CapturePoses (the pose packet of all hands of the pawn)
  -> Server RPC on the pawn's interactor
  -> ExecuteAction: interval, AcceptPoses, repeated TraceTarget, CanBeginGrip, TryClaim
  -> BeginGrip | Interaction | Use | EndGrip
  -> HeldGrip (replication) + ClientActionResult -> OnActionResult
```

### Poses and item motion

```text
Client: the first active interactor -> ServerUpdatePoses (PoseReplicationHz), each hand has its own sample number
Server: AcceptPoses -> validation of each sample separately -> server copies of the hands
Server: manipulation Tick (after the hands) -> SolveDesired -> SetWorldTransform or PhysicsHandle
       -> State (quantized pose, push model) -> replication
Clients: smooth approach to State; the owner solves the pose themself (Direct) or predicts it (Physics)
Rest: tick once per IdleUpdateInterval; motion or the body waking -> every frame again
```

### Linked actions

The bow reserves the arrow through `ReserveLinked`, becomes its external driver (`SetExternalDriver`) and sets the pose every tick (`SetExternalPose`). An intentional release of the arrow computes the velocity on the server, turns on simulation and calls `OnShot`. The docking support attaches a free part through `AttachToSurface`; a repeated grab removes the attachment.

## Invariants

- **One ledger of rights.** Only `UInteractionClaimSubsystem` decides who holds an item. World components do not start their own registries: manipulation, the bow and the support use one claim graph. External systems may read the claimant but must not change ownership bypassing `TryClaim`/`ReleaseClaim`.
- **One motion "writer".** The pose, scale, attachment and simulation of an item are changed only by its manipulation. That is why it turns off the owner's `ReplicateMovement`, requires a non-replicated root (or `UInteractionMeshComponent`), and `AInteractionPhysicsActor` ignores `OnRep_ReplicatedMovement` and `OnRep_AttachmentReplication`.
- **The server decides gameplay.** The client shows focus, but the right to act, physics, damage and the shot are decided on the server.
- **A throw is only intentional.** Only the reason `Released` passes velocities. Loss of tracking, cancellation, recenter and an invalid target release the item without a throw; a bow does not shoot without an intentional release.
- **Request identity.** `HeldGrip` stores the target together with `RequestId`, so late answers do not cancel new requests.
- **Weak references.** The ledger and the local focus sets do not extend the lifetime of actors and components.

## Extension points

| You need | How |
|---|---|
| A door, button, switch | `IInteractableInterface` in Blueprint or C++ ([11](11-Blueprint-Integration.md), [12](12-Cpp-Integration.md)) |
| An item taken with hands | `UInteractionManipulationComponent` on any actor or a subclass of `AInteractionPhysicsActor` |
| Different holding behavior | A Blueprint subclass of `UInteractionBlueprintGrabTargetComponent` or a C++ subclass of `UInteractionGrabTargetComponent` |
| A message to the player about a rejection | `OnActionResult` with `EInteractionRejectReason` |
| A reaction to grab, use, release, handle travel | The manipulation events `OnGripStarted`, `OnGripEnded`, `OnUsed`, `OnHandleChanged` |
| An item moves together with another actor | `SetExternalDriver` + `SetExternalPose` |
| Custom indication | The interactor's `OnFocusChanged` or the target's `OnFocusStart`/`OnFocusEnd`; `AInteractionMasterItem` is an example |
| A custom reaction to a physics actor's impact | Override `OnComponentHitInternal` in C++ |
| An edge strike | `UInteractionStrikeComponent::OnStrike` |

## Update order

Everything runs on the game thread, without asynchronous tasks.

| Who | When it ticks |
|---|---|
| `UInteractorComponent` | `TG_PrePhysics`, every frame while active |
| `UInteractionManipulationComponent` | `TG_PrePhysics`; every frame while there is work, otherwise once per `IdleUpdateInterval` (0.5 s); during a grip — after the hands that hold it |
| `UInteractionBowComponent` | `TG_PrePhysics`, after the bow's manipulation; the arrow search once per 0.05 s |
| `UInteractionStrikeComponent` | `TG_PostPhysics`, only on the server, while the item is held; frames without item motion are skipped |
| `UInteractionDockComponent` | Once per 0.1 s on the server |
| `AInteractionMasterItem` | Once per 0.1 s, not on a dedicated server |
| `AInteractionSpearActor` | Only in flight, on the server |

## What is not part of the plugin

Locomotion and teleport, your game's input mappings (the demo has its own), inventory, hand animation or IK, UI widgets (except the demo), the damage and health model (except the simple World reactions), saving state. These parts are left to the game: the plugin provides the extension points above.
