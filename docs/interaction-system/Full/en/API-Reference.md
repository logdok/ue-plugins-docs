# API reference

*🇬🇧 English | [🇺🇦 Українська](../uk/API-Reference.md)*

This page lists the public classes of InteractionSystem 1.0: what they do, what properties and default values they have, where to call functions and on which machine events fire. Distances are in centimeters, angles in degrees, velocities in cm/s and degrees/s.

Markers in the tables:

- **BP** — available in Blueprint (in brackets: `RW` — read and write, `RO` — read only);
- **C++** — C++ only;
- **Server** — the function works only on the authority and does nothing on a client;
- **Local** — the event or function runs only on the machine of the player who controls the pawn.

| Header | Module | Contents |
|---|---|---|
| `InteractorComponent.h` | `InteractionSystem` | `UInteractorComponent`, the focus and result delegates |
| `InteractableInterface.h` | `InteractionSystem` | `IInteractableInterface` |
| `InteractionSystemTypes.h` | `InteractionSystem` | `EInteractionAction`, `EInteractionRejectReason`, `FInteractionFilter`, `FInteractionRules` |
| `InteractionSystemSettings.h` | `InteractionSystem` | `UInteractionSystemSettings` — the project's network limits |
| `InteractionGripTypes.h` | `InteractionSystem` | `EInteractionHand`, `EInteractionReleaseReason`, `FInteractionGripMath` |
| `InteractionGrabTargetComponent.h` | `InteractionSystem` | `UInteractionGrabTargetComponent` |
| `InteractionBlueprintGrabTargetComponent.h` | `InteractionSystem` | `UInteractionBlueprintGrabTargetComponent` |
| `InteractionClaimSubsystem.h` | `InteractionSystem` | `UInteractionClaimSubsystem` |
| `InteractionGameplayTags.h` | `InteractionSystem` | `InteractionGameplayTags::Item`, `Locked` |
| `InteractionManipulationComponent.h` | `InteractionSystemWorld` | `UInteractionManipulationComponent` |
| `InteractionManipulationProfile.h` | `InteractionSystemWorld` | `FInteractionManipulationSettings`, `UInteractionManipulationProfile`, the motion enums |
| `InteractionGripPointComponent.h` | `InteractionSystemWorld` | `UInteractionGripPointComponent` |
| `InteractionDockComponent.h` | `InteractionSystemWorld` | `UInteractionDockComponent` |
| `InteractionBowComponent.h` | `InteractionSystemWorld` | `UInteractionBowComponent` |
| `InteractionStrikeComponent.h` | `InteractionSystemWorld` | `UInteractionStrikeComponent` |
| `InteractionDestructibleComponent.h` | `InteractionSystemWorld` | `UInteractionDestructibleComponent` |
| `InteractionMeshComponent.h` | `InteractionSystemWorld` | `UInteractionMeshComponent` |
| `InteractionPhysicsActor.h` | `InteractionSystemWorld` | `AInteractionPhysicsActor` |
| `InteractionMasterItem.h` | `InteractionSystemWorld` | `AInteractionMasterItem` |
| `InteractionSpearActor.h` | `InteractionSystemWorld` | `AInteractionSpearActor` |
| `InteractionPhysicsDataAsset.h` | `InteractionSystemWorld` | `UInteractionPhysicsDataAsset` |
| `InteractionDebugLibrary.h` | `InteractionSystemDebug` | `UInteractionDebugLibrary` |
| `InteractionCheatManagerExtension.h` | `InteractionSystemDebug` | Debug console commands |

## Core: InteractionSystem

### UInteractorComponent

A scene component on a pawn: it searches for a target, shows local focus and sends actions to the server. One component is one action source: a camera on desktop or one VR hand. The component replicates; RPCs go through the pawn owned by the client.

| Property | Default | Access | Purpose |
|---|---|---|---|
| `bUsePawnViewpoint` | `true` | BP RW | Move the component to the controller's view point every frame. Turn off for a VR hand |
| `TraceDistance` | 300 | BP RW | Ray length |
| `TraceRadius` | 0 | BP RW | 0 — line trace; above 0 — sphere sweep with a contact visibility check |
| `FocusUpdateHz` | 30 | BP RW | Focus search rate of this source |
| `PoseReplicationHz` | 30 | BP RW | Rate of the shared pose packet (1–120); the value of the pawn's first active interactor applies |
| `bNearInteraction` | `false` | BP RW | Search for a target with an overlap sphere around the palm (VR) instead of a ray |
| `NearRadius` | 15 | BP RW | Near search radius |
| `Hand` | `Any` | BP RW | The hand side for authored grip points |
| `PalmOffset` | Identity | BP RW | Palm calibration relative to the component |
| `TrackingLossGrace` | 0.25 s | BP RW | How long to tolerate tracking loss before releasing the item without a throw |
| `TraceChannel` | `Visibility` | BP RW | The channel for search and obstacle checks; targets must block it |
| `Filter` | empty | BP RW | Target tag requirements, see [FInteractionFilter](#finteractionfilter) |
| `GrabOffset` | (0, 0, 0) | BP RO | Local offset of the held item from the source; zero for VR |
| `MaxOriginDistance` | 500 | BP RO | The largest distance of a reported pose from the pawn |
| `MinActionInterval` | 0.08 s | BP RO | The minimum interval between actions the server accepts |
| `PoseTimeout` | 0.75 s | BP RO | After this long without any pose sample the server releases the item of a remote source |
| `MaxThrowSpeed` | 2500 | BP RO | The server clamps the linear throw velocity |
| `MaxAngularSpeed` | 3600 | BP RO | The server clamps the angular throw velocity |

The five network limits apply only when the corresponding flag `bOverrideMaxOriginDistance`, `bOverrideMinActionInterval`, `bOverridePoseTimeout`, `bOverrideMaxThrowSpeed` or `bOverrideMaxAngularSpeed` is on; otherwise the component takes the value from [UInteractionSystemSettings](#uinteractionsystemsettings).

| Function | Access | Where to call | Action |
|---|---|---|---|
| `RequestGrab()` | BP | Local | Request to grab the current target |
| `RequestRelease(Linear, Angular)` | BP | Local | Release the item; zero vectors — no throw |
| `RequestInteraction()` | BP | Local | An ordinary action with the target; if the hand holds an item — its `Use` |
| `CancelGrip(Reason)` | BP | Local or server | Safely end the grip without a throw; `Released` is replaced with `Cancelled` |
| `SetTrackingValid(bValid)` | BP | Local, every frame | Report whether the device pose is reliable |
| `RefreshFocus()` / `ClearFocus()` | BP | Local | Update or clear focus immediately |
| `GetFocusedActor()` | BP | Local | The current focus target or `None` |
| `GetHeldActor()` | BP | Anywhere | The held actor confirmed by the server |
| `CanFocusNewTarget()` | BP | Local | The hand is active, tracked and free |
| `IsLocallyControlled()` | BP | Anywhere | The pawn is controlled on this machine |
| `GetGripTransform()` | BP | Anywhere | The calibrated palm pose with unit scale |
| `HasRequestedGrip()` | BP | Local | There is a held item or a request waiting for an answer |
| `HasValidTracking()` | BP | Anywhere | The current tracking state of the source |
| `GetMaxOriginDistance()`, `GetMinActionInterval()`, `GetPoseTimeout()`, `GetMaxThrowSpeed()`, `GetMaxAngularSpeed()` | BP | Anywhere | The effective limit: the component's override or the project value |
| `TraceTarget(Origin, Rotation, Action)` | C++ | Anywhere | Target search by the same rules as focus and the server |
| `CancelTarget(Target, Reason)` | C++ | Server | Release exactly this target, if the hand holds it |

| Event | Where it fires | Parameters |
|---|---|---|
| `OnFocusChanged` | Local | `PreviousActor`, `NewActor` |
| `OnActionResult` | On the client that sent the request and on a local authority | `TargetActor`, `Action`, `bAccepted`, `Reason` — the [rejection reason](#core-enums) |

### IInteractableInterface

The target interface for an ordinary action and local feedback. Implement it in Blueprint (**Class Settings → Interfaces**) or C++.

| Function | Where it runs | Default implementation | Responsibility |
|---|---|---|---|
| `CanInteract(Interactor, Action)` | Client and server | `true` | Allow the action; without side effects |
| `GetInteractionTags()` | Client and server | an empty container | Gameplay Tags for the filter |
| `Interaction(Interactor)` | Server | — | The gameplay action: a door, a button, a switch |
| `OnFocusStart(Interactor)` / `OnFocusEnd(Interactor)` | Local | — | Visual feedback for a specific hand |
| `OnGrab(Interactor)` | Server | — | The core calls it when the first hand takes the target |
| `OnRelease(Interactor, Reason, LinearVelocity, AngularVelocity)` | Server | — | The core calls it when the last hand lets go; velocities are non-zero only for `Released` |

`Interaction`, `OnFocusStart`, `OnFocusEnd`, `OnGrab` and `OnRelease` are events that you implement but do not call from Blueprint: this way a client cannot bypass the server and the claim ledger. `CanInteract` and `GetInteractionTags` can be called.

### FInteractionFilter

| Field | Meaning |
|---|---|
| `RequiredTags` | The target must contain all these Gameplay Tags (with hierarchy) |
| `BlockedTags` | Any of these tags forbids the target |
| `RequiredActorTags` | All these names must be in `Actor.Tags` (exact match) |

`Matches(Tags, ActorTags)` returns `true` if all requirements are met. Empty lists restrict nothing.

### FInteractionRules

Static checks without touching the world, shared by the component and the tests:

| Function | Returns `true` if |
|---|---|
| `IsPoseInReach(Origin, Rotation, PawnLocation, MaxOffset)` | The values are finite and `Origin` is no farther than `MaxOffset` from the pawn |
| `IsFiniteVelocity(Linear, Angular)` | Both velocities are finite |

### UInteractionGrabTargetComponent

An abstract boundary between the interactor and the way an item moves. The ready-made implementation is [UInteractionManipulationComponent](#uinteractionmanipulationcomponent); the implementation with rules in Blueprint is [UInteractionBlueprintGrabTargetComponent](#uinteractionblueprintgrabtargetcomponent). A custom implementation in C++ is shown in [12](12-Cpp-Integration.md#custom-grab-target).

| Member | Access | Purpose |
|---|---|---|
| `TargetTags` | BP RW | Tags for the filter if the actor does not implement the interface |
| `CanBeginGrip(Hand) const` | C++, virtual | Whether a grip can begin; called both locally for focus and on the server. `false` by default |
| `GetMaximumHands() const` | C++, virtual | How many hands of one pawn the target allows. 1 by default |
| `BeginGrip(Hand)` | C++, virtual, server | After reservation; `false` cancels the reservation. `false` by default |
| `EndGrip(Hand, Reason, Linear, Angular)` | C++, virtual, server | Remove only this hand; a forced ending must not throw the item |
| `Use(Hand)` | C++, virtual, server | The action of the hand that holds the item. `false` by default |

### UInteractionBlueprintGrabTargetComponent

A grab target whose rules are set by a Blueprint subclass of this component. The component itself checks `bGrabEnabled`, free space and that the second hand belongs to the same pawn, and then asks your events. The owner's motion is performed by your Blueprint. The component replicates.

| Member | Access | Purpose |
|---|---|---|
| `MaximumHands` | BP RO, 1–2 | How many hands of one pawn may hold the owner |
| `bGrabEnabled` | BP RO, replicated | Whether new grips are accepted; change it through `SetGrabEnabled` |
| `SetGrabEnabled(bEnabled)` | BP, server | Turn grabbing on or off; turning it off releases the current hands with `Cancelled` |
| `CancelAllGrips()` | BP, server | Release all hands with `Cancelled` |
| `GetHands()` / `GetHandCount()` | BP, replicated | The hands that hold the owner, in the order of grabbing |
| Event `Can Begin Grip(Hand)` | Client and server | An extra admission rule; replicated state only. `true` by default |
| Event `Begin Grip(Hand)` | Server | The hand is admitted and reserved; `false` cancels the grip (`Unavailable`). `true` by default |
| Event `End Grip(Hand, Reason, Linear, Angular)` | Server | The hand released the owner |
| Event `Use(Hand)` | Server | The hand holding the owner pressed "use"; `true` — handled. `false` by default |

### UInteractionSystemSettings

**Project Settings → Plugins → Interaction System** (`Config/DefaultGame.ini`, section `[/Script/InteractionSystem.InteractionSystemSettings]`). The default network limits for all `UInteractorComponent`s, unless a component overrides the value.

| Field | Default | Purpose |
|---|---|---|
| `MaxOriginDistance` | 500 cm | The largest distance of a reported pose from the pawn |
| `MinActionInterval` | 0.08 s | The minimum time between accepted requests of one source |
| `PoseTimeout` | 0.75 s | How long to wait for any pose sample of a remote source |
| `MaxThrowSpeed` | 2500 cm/s | The largest linear throw velocity |
| `MaxAngularSpeed` | 3600 degrees/s | The largest angular throw velocity |

### UInteractionClaimSubsystem

A world subsystem — the server ledger of the right to a target. Queries are available in Blueprint, changes only in C++; on a client the ledger is empty.

| Function | Purpose |
|---|---|
| `TryClaim(Target, Interactor, MaximumHands)` | Add a hand to the grip of the same pawn within the limit (1 or 2) |
| `ReleaseClaim(Target, Interactor)` | Remove only this hand |
| `FindClaimant(Target)` / `FindHands(Target)` | BP. The first hand or all hands |
| `IsClaimed(Target)` | BP. There is a hand or a linked reservation |
| `FindOwningPawn(Target)` | BP. The pawn that owns the grip or reservation |
| `GetClaimedTargets()` | BP. All targets that are held or reserved |
| `FindLinkedParent(Target)` | BP. The parent item of a linked reservation, for example the bow for an arrow |
| `ReserveLinked(Target, Parent, Pawn)` | Link a dependent target (an arrow) with the held parent of the same pawn; cycles are forbidden |
| `ReleaseLinked(Target, Parent)` | Remove a linked reservation |
| `ForgetTarget(Target)` | Forget the target and recursively its dependent reservations |

Usually reading the owner is enough; change the ledger only through `RequestGrab`/`RequestRelease`.

### FInteractionGripMath

Deterministic grip math that does not touch the world: `MakePairFrame` builds the coordinate system of a pair of palms, `Capture` fixes the baseline, `Solve` computes the pose for one or two hands with scale and pivot, `PlaneAngle` returns the angle of a point around an axis. It is used by the manipulation and the tests.

### Core enums

| Enum | Values |
|---|---|
| `EInteractionAction` | `Interact` — use; `Grab` — take; `Release` — let go |
| `EInteractionHand` | `Any` — no side (desktop); `Left`; `Right` |
| `EInteractionReleaseReason` | `Released` — the player let go (only this reason allows a throw); `TrackingLost`; `Cancelled`; `TargetInvalid`; `Recentered` |
| `EInteractionRejectReason` | `None` — accepted; `Cooldown`; `InvalidTarget`; `InvalidPose`; `NotReachable`; `HandBusy`; `Claimed`; `Unavailable`. The meanings and the order of checks are in [13](13-Network-Model.md#rejection-reasons) |

### InteractionGameplayTags

| C++ variable | Tag | Purpose |
|---|---|---|
| `InteractionGameplayTags::Item` | `Interaction.Item` | Marks an item for `RequiredTags` |
| `InteractionGameplayTags::Locked` | `Interaction.Locked` | The typical tag for `BlockedTags` |

## World: InteractionSystemWorld

### UInteractionManipulationComponent

A ready-made grab target for any actor: one or two hands, authored points, free grip, scale, physical holding, rotary and linear handles, including for a named part of an actor. The component replicates one atomic snapshot (hands, grip baseline, quantized pose with scale, the lag of the physical body, attachment, external driver, handle travel), supports push model and on the server turns off the owner's `ReplicateMovement`. At rest the component ticks rarely and wakes on motion.

| Property | Default | Access | Purpose |
|---|---|---|---|
| `Profile` | `None` | BP RW | A shared profile; if empty — `InlineSettings` |
| `InlineSettings` | see below | BP RW | Settings of this instance |
| `TargetComponentName` | `None` | BP RW | The name of the component to move; `None` — the actor's root. A named part stays under its parent |
| `bStartWithPhysics` | `false` | BP RW | Simulation on the server from the start of the game |
| `bGrabEnabled` | `true` | BP RW, replicated | Turning it off rejects new grips and ends the current ones; change it on the server or through `SetGrabEnabled` |
| `RemoteInterpolationSpeed` | 20 | BP RW | Snapshot smoothing on clients; 0 — no smoothing |
| `bPredictPhysicsForOwner` | `true` | BP RW | Predict a `Physics` item in the owner's hand |
| `OwnerPredictionTolerance` | 10 | BP RW | The lag of the server body after which the owner sees the server pose |
| `IdleUpdateInterval` | 0.5 s | BP RW | The tick interval at rest; motion and the body waking bring back the every-frame tick |

| Function | Access | Where to call | Action |
|---|---|---|---|
| `GetHandCount()` | BP | Anywhere | The number of hands holding the item |
| `GetPrimaryHand()` / `GetSecondaryHand()` | BP | Anywhere | The primary and supporting hands |
| `GetManipulatedComponent()` | BP | Anywhere | The component the manipulation moves |
| `ValidateSetup(OutReason)` | BP | Anywhere | Validate the profile and target; the reason for failure is in `OutReason` |
| `GetHandleValue()` | BP | Anywhere | The handle travel: degrees for `Rotary`, centimeters for `Linear` |
| `IsExternallyDriven()` | BP | Anywhere | The motion is controlled by an external driver |
| `CancelAllGrips()` | BP, authority only | Server | End all grips without a throw |
| `ResetManipulation()` | BP, authority only | Server | Cancel grips, the driver and attachment; return a free item to `HomeTransform`, a handle to travel 0, a named part under its parent |
| `SetGrabEnabled(bEnabled)` | BP, authority only | Server | Allow or forbid grabbing; turning it off releases the current hands |
| `GetHomeTransform()` / `SetHomeTransform(Transform)` | BP; writing — authority only | Anywhere / server | The home pose for `ResetManipulation`; remembered in `BeginPlay` |
| `SyncTransform()` | BP, authority only | Server | Immediately send clients a pose changed by gameplay code |
| `SetExternalDriver(Driver)` | BP, authority only | Server | Hand the motion to another actor; `None` returns ordinary manipulation |
| `SetExternalPose(Pose)` | BP, authority only | Server | The item's pose under an external driver |
| `GetSettings()` | C++ | Anywhere | The effective settings (on a client — from the snapshot) |
| `AttachToSurface(Parent, Socket)` | BP, authority only | Server | Attach a free item to a component, preserving its world pose |
| `SetFreeSimulation(bSimulate, Velocity, AngularVelocity)` | BP, authority only | Server | For a free item: remove attachment and driver, turn simulation on or off with velocities |
| `IsAttached()` | BP | Anywhere | The item is attached to a surface |

| Event | Where it fires | Parameters |
|---|---|---|
| `OnGripStarted` | Server | `Hand`, `bFirst` — `true` if the item was free before |
| `OnGripEnded` | Server | `Hand`, `bLast`, `Reason`, `LinearVelocity`, `AngularVelocity` |
| `OnUsed` | Server | `Hand` — the hand that holds the item and called `RequestInteraction` |
| `OnHandleChanged` | Server | `Value`, `Fraction` 0..1; also `(0, 0)` after `ResetManipulation` |

### FInteractionManipulationSettings

| Field | Default | Purpose |
|---|---|---|
| `Movement` | `Direct` | `Direct` — an exact pose every frame; `Physics` — a server `PhysicsHandle` with collisions |
| `GripSelection` | `Free` | `Free`, `Authored` — only grip points, `PreferAuthored` — a point if nearby, otherwise free |
| `Alignment` | `KeepOffset` | `KeepOffset` — no jump; `Align` — align the point (or the root) with the palm |
| `MaximumHands` | 2 | 1 or 2 hands of one pawn |
| `TwoHandPivot` | `Primary` | The primary hand as the anchor or `Midpoint` between the hands |
| `bAllowScaling` | `false` | Uniform scale with two hands (only `Direct`, two hands, no constraint) |
| `MinimumScale` / `MaximumScale` | 0.25 / 4 | Scale limits along the largest axis |
| `bSimulateOnRelease` | `true` | Turn physics on after the last hand |
| `bAllowThrow` | `true` | Pass velocities on an intentional release |
| `LinearStiffness` / `LinearDamping` | 3000 / 200 | The position drive for `Physics` |
| `AngularStiffness` / `AngularDamping` | 2500 / 200 | The rotation drive for `Physics` |
| `BreakDistance` | 100 | How far the body may lag behind the desired pose beyond its closest position after the grab; beyond that the grip is cancelled |
| `Constraint` | `None` | `Rotary` or `Linear` (only `Direct`, one hand, no scale) |
| `ConstraintAxis` | (1, 0, 0) | The axis in the target's initial coordinate system |
| `MinimumValue` / `MaximumValue` | 0 / 720 | Travel limits: degrees or centimeters |

The ready-made subclasses change some values: `AInteractionPhysicsActor` — `MaximumHands=1`, `AInteractionMasterItem` — `Alignment=Align`.

### UInteractionManipulationProfile

A Data Asset (**Content Browser → Add → Miscellaneous → Data Asset → InteractionManipulationProfile**) with one field `Settings` of type `FInteractionManipulationSettings`. A profile is shared by many items and does not change during play.

### UInteractionGripPointComponent

| Property | Default | Purpose |
|---|---|---|
| `Source` | `Component` | `Component` — the point's own transform; `Socket` — a mesh socket |
| `SocketComponentName` | `None` | The component with the socket; `None` — the point's parent component |
| `SocketName` | `None` | The socket name |
| `SocketOffset` | Identity | Correction relative to the socket |
| `bFallbackToComponent` | `false` | Allow the point's transform if the socket is missing |
| `Radius` | 20 | The maximum distance from the palm to the point |
| `AllowedHand` | `Any` | Restriction to a hand side |
| `bAllowFirstHand` / `bAllowSecondHand` | `true` / `true` | Whether the point may start a grip / add a second hand |
| `Priority` | 0 | A higher priority wins; among equals — the nearest point |

`ResolveAnchor(OutTransform)` (BP) returns the point's world pose or `false` if the socket is unavailable.

### UInteractionDockComponent

A docking support for a single part. Every 0.1 s the server looks for a free part nearby.

| Property | Default | Purpose |
|---|---|---|
| `RequiredPartTag` | `Interaction.Part` | The Actor Tag without which a part will not sit |
| `CaptureRadius` | 12 | The distance from the part's origin to the support |
| `AlignmentTolerance` | 35° | The permitted orientation error |

A part sits if it has a `UInteractionManipulationComponent`, nobody holds or reserves it, it is not attached or under an external driver, and there is no obstacle on `Visibility` between it and the support. Then the support turns the simulation off, puts the part into its own transform (keeping the part's scale) and attaches it. A repeated grab frees the place. `GetOccupant()` (BP) returns the part, `TryDock(Part)` (BP, server) tries to seat it immediately, for example for an initially assembled state.

### UInteractionBowComponent

A bow with a linked arrow: one hand holds the bow, another hand of the same pawn holds the arrow.

| Property | Default | Purpose |
|---|---|---|
| `ArrowTag` | `Interaction.Arrow` | The arrow's Actor Tag |
| `StringRestPoint` | `None` | A component or grip point with the string's pose; `None` — `StringRestLocal` |
| `StringRestLocal` | (-15, 0, 0) | The string's pose relative to the bow's root; the X axis points to the target |
| `ArrowGuideLocal` | (15, 0, 0) | The point the arrow passes through |
| `ArrowNockPoint` | `None` | A component or point of the arrow's tail; `None` — the arrow's root |
| `NockRadius` | 18 | The distance from the tail to the string for nocking |
| `MaximumDraw` | 55 | The largest draw |
| `MinimumShotFraction` | 0.1 | A smaller draw returns the string without a shot |
| `MaximumLateralOffset` | 35 | The sideways deviation of the hand that cancels the nocking |
| `FullDrawSpeed` | 2200 | The speed at full draw |
| `DrawExponent` | 1 | The curve `Speed = FullDrawSpeed × Draw^DrawExponent` |

| Member | Access | Purpose |
|---|---|---|
| `GetDrawFraction()` | BP, replicated | The draw 0..1 for the string animation |
| `GetNockedArrow()` | BP, replicated | The nocked arrow |
| `TryNock(Arrow)` | BP, server | Nock a held compatible arrow |
| `CancelNock()` | BP, server | Take off the arrow without a shot |
| `OnShot` | Event, server | `Projectile`, `Velocity` — after an intentional release of the arrow |

Before `OnShot` the component itself turns on the arrow's simulation with the computed velocity. For `AInteractionSpearActor` call `Launch(Velocity)` in the handler to turn on flight and embedding.

### UInteractionStrikeComponent

The edge-strike event of a held item; the damage is decided by the game.

| Property | Default | Purpose |
|---|---|---|
| `EdgeStart` / `EdgeEnd` | (30, -5, 0) / (30, 5, 0) | The edge in the local coordinates of the manipulation target |
| `EdgeRadius` | 2 | The sweep thickness |
| `MinimumSpeed` | 120 | The minimum speed of an edge point |
| `RequiredTargetTag` | `None` | The target's Actor Tag; `None` — any |

`OnStrike(Target, Hit, Speed)` fires on the server once per contact episode, only while the item is held and the edge moves toward the surface. The speed is computed over the time between two poses of the item, so frames without a new pose of the remote hand do not inflate it. A `UInteractionManipulationComponent` is required on the same actor.

### UInteractionDestructibleComponent

| Member | Access | Purpose |
|---|---|---|
| `Health` | Class Defaults | The strength reserve, 50 |
| `DamagePerImpulse` | Class Defaults | Damage per unit of impact impulse, 0.01 |
| `MinimumImpactDamage` | Class Defaults | An impact counts if its damage is greater than this value (5) |
| `DestroyEffect` | Class Defaults | The effect at the destruction point: Niagara or Cascade |
| `GetHealth()` | BP | The remaining strength on the server |
| `ApplyImpactDamage(Amount, Location)` | BP, server | Deal damage; at zero — the effect and destruction of the owner |
| `OnImpactDestroyed(Location)` | Event, server and all clients | Before the owner is destroyed: sound, debris, a custom effect |

### UInteractionMeshComponent

A `UStaticMeshComponent` with transform and attachment replication turned off. It can be made replicated (mesh, materials, visibility) without creating a second motion "writer" next to the manipulation snapshot. An ordinary non-replicated static mesh works too.

### AInteractionPhysicsActor

A ready-made replicated item: the root `MeshComponent` (`UInteractionMeshComponent`), `Manipulation` and `IInteractableInterface`.

| Member | Access | Purpose |
|---|---|---|
| `ObjectData` | BP RO | [UInteractionPhysicsDataAsset](#uinteractionphysicsdataasset) |
| `bInteractionEnabled` | BP RW, replicated | Turns off both grabbing and the ordinary action |
| `bCanBeGrabbed` | BP RW, replicated | Grab permission; doors and buttons — `false`. Change it on the server |
| `InteractionTags` | BP RW | Gameplay Tags for the filter |
| `OnInteracted` | Event, server | A successful ordinary action; the parameter is the interactor |
| `IsHeld()` | BP | At least one hand holds the item |
| `OnComponentHitInternal(...)` | C++, protected virtual | The reaction to a free impact: sound and, if enabled, damage, see [05](05-Physics.md#impacts-sound-and-damage) |
| `OnGrab` / `OnRelease` | Interface implementation | Remember the damage instigator; start the `DisablePhysicsFixedTime` timer. In a subclass call the parent implementation |

`CanInteract` allows `Grab` if `bCanBeGrabbed`, and `Interact` if nobody holds the item.

### AInteractionMasterItem

A physics item with local indication: `ProximitySphere` (350 cm, overlap with Pawn), the text markers `NearbyMarker` ("•") and `FocusMarker`, optional billboards `NearbyIcon` and `FocusIcon`, mesh highlighting.

| Property | Default | Purpose |
|---|---|---|
| `FocusOverlayMaterial` | `None` | The overlay material during focus |
| `bUseCustomDepthFocus` | `true` | Write the mesh into custom depth for an outline |
| `FocusStencil` | 252 | The stencil value for your post-process |
| `InteractionPrompt` | `E` | The focus prompt text |

`HasLocalFocus()` (BP) shows whether at least one free local hand focuses the item. The indication updates 10 times a second and is turned off on a dedicated server.

### AInteractionSpearActor

| Member | Default | Purpose |
|---|---|---|
| `PenetrationDepth` | 15 | How deep the tip sinks after a hit |
| `TipContactLength` | 40 | The length of the mesh's front part that can embed |
| `MinimumEmbedSpeed` | 200 | The minimum speed for embedding |
| `Launch(Velocity)` | BP, server | Start flight without a hand (an arrow from a bow) |
| `GetSpearTipLocation()` | BP | The tip's world position (+X, `Bounds.Max.X`) |
| `IsEmbedded()` | BP | The spear is embedded in a surface |

### UInteractionPhysicsDataAsset

| Field | Default | Purpose |
|---|---|---|
| `MassInKg` | 1 | The body's mass |
| `bStartWithPhysics` | `false` | Simulation from the start of the game |
| `PostThrowBehavior` | `SleepWhenSettled` | `SleepWhenSettled` or `DisablePhysicsFixedTime` |
| `PhysicsDisableDelay` | 5 s | The delay for `DisablePhysicsFixedTime` |
| `bAlignToVelocity` | `true` | The spear turns its tip along the motion |
| `AlignmentStiffness` | 5 | The speed of that turn |
| `HitSound` | `None` | The impact sound |
| `HitSoundMinimumImpulse` | 150 | The smallest impulse that plays the sound |
| `HitSoundCooldown` | 0.15 s | The shortest gap between sounds of one item |
| `bApplyImpactDamage` | `false` | Deal damage to actors a free item hits |
| `DamagePerImpulse` | 0.005 | Damage per unit of impulse |
| `MinimumDamageImpulse` | 20 | Weaker impacts deal no damage |
| `DamageTypeClass` | `UDamageType` | The damage type for `ApplyDamage` |

### World enums

| Enum | Values |
|---|---|
| `EInteractionMovement` | `Direct`, `Physics` |
| `EInteractionGripSelection` | `Free`, `Authored`, `PreferAuthored` |
| `EInteractionGripAlignment` | `KeepOffset`, `Align` |
| `EInteractionTwoHandPivot` | `Primary`, `Midpoint` |
| `EInteractionConstraint` | `None`, `Rotary`, `Linear` |
| `EInteractionAnchorSource` | `Component`, `Socket` |
| `EInteractionPostThrowBehavior` | `SleepWhenSettled`, `DisablePhysicsFixedTime` |

## Demo: InteractionSystemDemo

The demo classes are examples of using the public API. They can be browsed and inherited from, but you do not have to ship them in your game.

| Class | Map | Purpose |
|---|---|---|
| `AInteractionDemoCharacter`, `AInteractionDemoHUD`, `AInteractionDemoGameMode` | InteractionLab | The desktop character: E — take or use, F — throw; creates the Enhanced Input actions in code |
| `AInteractionDemoDoor` | InteractionLab | A door on `AInteractionPhysicsActor` with `bCanBeGrabbed=false` |
| `AInteractionDemoSpear`, `AInteractionSpearDemoCharacter`, `AInteractionSpearDemoGameMode` | SpearRange | A spear with local indication and a character for throwing |
| `AInteractionVRPawn`, `AInteractionVRHUD`, `AInteractionVRGameMode` | VRLab, VRLabBP | An OpenXR pawn with two hands and a desktop emulator |
| `AInteractionVRItem`, `AInteractionVRAxe`, `AInteractionVRBow`, `AInteractionVRArrow`, `AInteractionVREngine`, `AInteractionVRPanel` | VRLab | The station items |
| `AInteractionVRBunkerDoor`, `AInteractionVRChoppingTarget` | VRLab | The door with a wheel and the log |
| `UInteractionVRPanelWidget` | VRLab, VRLabBP | The native widget of the spatial panel |
| `UInteractionVRBowVisualLibrary::StringSegmentTransform` | VRLab, VRLabBP | The pose of a string section by the draw |

`AInteractionVRPawn` has `bForceEmulation`, `InputContext`, `IsEmulating()`, `RequestLabReset()` and the Blueprint event `OnLabResetRequested`, which VRLabBP uses.

## Debugging: InteractionSystemDebug

A leaf module of debug tools ([15](15-Debug-Tools.md)). It registers nothing in Shipping builds.

| Member | Access | Purpose |
|---|---|---|
| `UInteractionDebugLibrary::ToggleInteractionDebugOverlay(PlayerController)` | BP | Open or close the panel for a local player |
| `UInteractionDebugLibrary::IsInteractionDebugOverlayOpen(PlayerController)` | BP | Whether the panel is open |
| `InteractionOverlay`, `InteractionStatus`, `InteractionLog`, `InteractionClaims`, `InteractionRelease`, `InteractionResetAll` | Console | Cheat manager commands; described in [15](15-Debug-Tools.md#console-commands) |
