# 08 — VR manipulation and profiles

*🇬🇧 English | [🇺🇦 Українська](../uk/08-VR-Manipulation.md)*

This chapter describes the InteractionSystem 1.0 implementation for Unreal Engine 5.8. A single `UInteractionManipulationComponent` serves a rifle, an axe, binoculars, an engine, a spatial panel, a door wheel and any other actor. The profile — `UInteractionManipulationProfile` or `InlineSettings` — determines motion, number of hands, scaling, pivot, physics and constraints.

## How a profile is read

`MaximumHands=2` lets a second hand join only after the first and only from the same pawn. `GripSelection` has three modes: `Free` accepts the palm without an authored point, `Authored` requires a valid point, `PreferAuthored` uses a point if one is available and otherwise takes the free palm. `Alignment=Align` on the first grab aligns the authored anchor with the palm, and for a free grip the item's root with the palm; `KeepOffset` preserves the actual pose. In the simple InteractionLab the cubes use `Align` so that they end up near the camera after pressing E. In VR Lab the authored profiles with `KeepOffset` do not change the item's pose abruptly.

`Movement=Direct` applies the solved transform every frame. `Movement=Physics` creates one server `PhysicsHandle`; on release the profile can leave the body in simulation and pass the clamped velocities. The physics mode does not combine with scaling: this protects Chaos from a conflict between scale and physical mass. Other clients receive an atomic snapshot and smooth it with `RemoteInterpolationSpeed`.

> **In plain words.** `Direct` — the item is "glued" to the hand: where the hand is, the item is, even if the hand has passed through a wall. `Physics` — the item is on a spring: the hand pulls it and a wall stops it. For a sword that must press against a shield choose `Physics`; for binoculars choose `Direct`.

`BreakDistance` (100 cm) is counted from the closest position the body reached after the grab. An item taken from afar first "flies" to the hand, and the grip does not break. If the body is stuck while the hand has moved a further `BreakDistance` away, the grip ends safely.

The VR Lab profiles are in `/InteractionSystem/Demo/VRLab/Profiles` and are ready starting points:

| Profile | Key values | Demo items |
|---|---|---|
| `DA_RigidObject` | `Movement=Physics`, `MaximumHands=2`, `GripSelection=Authored`, `KeepOffset` | rifle, axe |
| `DA_SymmetricObject` | `MaximumHands=2`, `Authored`, `TwoHandPivot=Midpoint` | binoculars |
| `DA_InspectObject` | `MaximumHands=2`, `bAllowScaling`, `Midpoint`, no simulation or throw after release | engine |
| `DA_SpatialPanel` | `Authored`, `bAllowScaling`, `Midpoint`, scale 0.6–1.8, no simulation or throw | spatial panel |
| `DA_SingleHand` | `MaximumHands=1`, the rest default | piston, arrows |
| `DA_BowBody` | `MaximumHands=1`, `Authored`, no simulation or throw | bow |
| `DA_RotaryHandle` | `MaximumHands=1`, `Constraint=Rotary`, 0–720°, no simulation or throw | bunker wheel |

Default values of `FInteractionManipulationSettings`: `Direct`, `Free`, `KeepOffset`, `MaximumHands=2`, `Primary`, scale 0.25–4, `bSimulateOnRelease=true`, `bAllowThrow=true`, stiffness 3000/2500, damping 200/200, `BreakDistance=100` cm, `Constraint=None`, X axis, travel 0–720. The full list of fields is in the [API reference](API-Reference.md#finteractionmanipulationsettings). The profile is validated before every grip (`ValidateSetup`): scaling requires `Direct`, two hands and no constraint; a constraint requires `Direct` and one hand. An invalid profile does not let the item be taken, and an active grip ends safely.

The two-hand solver preserves distance, the pair frame, rotation and scale. `TwoHandPivot=Primary` keeps the primary hand as the anchor for a rifle; `Midpoint` scales and rotates around the midpoint for binoculars or a panel. The `MinimumScale` and `MaximumScale` limits apply symmetrically. When the hands nearly coincide, the solver temporarily holds the last valid pose and rebuilds the baseline after they separate — this produces no accidental 180° jump.

## Physics mode in the owner's hand

A player holding an item with `Movement=Physics` sees it in their hand at once, without network delay: the client predicts the pose with the same solver as the server. The server reports how far its body lags behind the desired pose (`BodySeparation`). While the lag is less than half of `OwnerPredictionTolerance` (10 cm by default), the owner sees their own prediction; beyond that they smoothly move to the server pose, so an item pressed against a wall stops in their hand as well. Other players always see the server pose.

| Property | Default | Purpose |
|---|---|---|
| `bPredictPhysicsForOwner` | on | Predict a physical item in the owner's hand |
| `OwnerPredictionTolerance` | 10 cm | The lag of the server body after which the owner sees the server pose |

## When the component updates

The component ticks every frame while an item is held, an external driver moves it, the body is simulated and awake, or a client is still moving the pose toward the server's. A resting item checks its state once per `IdleUpdateInterval` (0.5 s) and wakes at once when its transform changes or its physics body wakes up. So hundreds of items lying around barely load the game thread.

You can move an item from gameplay code with ordinary functions (`SetActorLocation`, `SetActorRotation`): the component notices the change and passes it to clients. `SyncTransform()` on the server sends the new pose immediately, without waiting for the next tick.

## Enabling and disabling grabbing

`SetGrabEnabled(false)` on the server forbids new grips and safely releases the current hands; `SetGrabEnabled(true)` allows taking again. The value of `bGrabEnabled` is replicated, so clients stop showing the prompt at once. This is how the bunker wheel in VR Lab becomes unavailable while the door is open.

## Returning to the home pose

`ResetManipulation()` on the server releases the hands and returns the item "home":

| Item | Where it returns |
|---|---|
| A free one | To `HomeTransform` — the pose at the start of the game |
| A handle with a constraint | To the zero travel of its constraint |
| A named part of an actor | Back under its parent, to the initial relative pose |

`HomeTransform` is remembered in `BeginPlay`. If an item was created and moved later, set the home on the server through `SetHomeTransform`; the current value is returned by `GetHomeTransform`.

## Sockets, components and their absence

`UInteractionGripPointComponent` has `Source=Socket` or `Source=Component`. For a socket anchor set `SocketName` and, if needed, `SocketComponentName` — the exact name of the component with this socket; an empty `SocketComponentName` means the parent component of the point itself. `SocketOffset` adjusts the pose relative to the socket without changing the mesh. For a component anchor a separate scene component with a local transform is enough. If a socket is renamed, the mesh is not loaded or the component is removed, the point becomes invalid and the grip ends without falling back to the origin. `bFallbackToComponent` lets you explicitly agree to a safe fallback.

The point is chosen by the highest `Priority`; among equals the one nearest the palm wins. The palm must be no farther than `Radius` (20 cm). `AllowedHand` restricts a point to the left or right hand, and `bAllowFirstHand` and `bAllowSecondHand` determine whether a point can start the grip or only add a supporting hand. A point already taken by another hand is not chosen again.

For an item without sockets choose `GripSelection=Free` and leave the authored points empty. The interactor takes the palm's `GetGripTransform()`; an animated skeletal hand, a hand mesh or a special socket is not needed. This is the same path for Quest 2 and the desktop emulator. Do not rely on an automatic origin: it is often inside the body and gives unexpected alignment.

An example for a rifle: `Grip_Main` on the `SM_Rifle` example is the first hand's socket, `Grip_Support` the second's. For the binoculars the demo uses two component points, so this scenario shows operation without sockets. For an arbitrary Blueprint it is enough to add a component, assign a profile and not change the core code.

## Linked items: the bow and the arrow

`UInteractionBowComponent` is not a separate kind of grab. The bow takes one hand; the second hand holds the arrow as an ordinary `UInteractionManipulationComponent`. When the arrow's tail enters `NockRadius`, the bow checks that both hands belong to the same pawn, calls `ReserveLinked(Arrow, Bow, Pawn)` and temporarily hands the arrow's motion to the external bow driver.

The demo bow has a solid curved silhouette of overlapping limbs and two visual string sections. Their ends coincide with points at the ends of the limbs `(-15, 0, ±65)` in local coordinates. The moving middle of the string stays tied to the current draw, but the visible geometry does not determine the shot physics.

The second hand moves relative to the current string anchor. A sideways departure beyond `MaximumLateralOffset` cancels the nocking. Releasing at less than `MinimumShotFraction` returns the string without a shot; an intentional release computes the velocity on the server and calls `OnShot`. Loss of tracking, `CancelGrip`, destroying the arrow or releasing the bow clears the linked reservation without a throw.

## Fixed handles and seatings

`Constraint=Rotary` accumulates a signed angle around `ConstraintAxis`, rejects tracking jumps of 90° and more and clamps the result to `MinimumValue..MaximumValue`. In VR Lab the red three-spoke wheel has a range of 0..720°, that is two full turns with the ability to re-grip the rim; after that the door opens and the wheel becomes externally driven together with the leaf. `Constraint=Linear` similarly projects the palm onto one axis; the travel is measured in world centimeters regardless of the component's scale.

### A movable part inside an actor

`TargetComponentName` lets you move not the whole actor but one of its components: a cabinet drawer, a chest lid, a console lever. Such a part stays under its parent both during the grip and after release. The constraint frame is tied to the parent: if the cabinet rides on a platform, the drawer rides with it and still slides out relative to the cabinet, and the accumulated travel is kept between grips. A step-by-step example is [lesson 4](14-Tutorials.md#lesson-4-a-drawer-that-slides-out-of-a-cabinet).

The part itself must be `Movable` and must not replicate separately (or must be a `UInteractionMeshComponent`): its motion is carried by the manipulation snapshot.

In VRLabBP the leaf has a separate `DoorPanel` component, offset from the root to the middle of the door: the root remains the side axis of the hinge. After unlocking, the Blueprint releases the wheel's grip with `CancelAllGrips`, assigns the door as the external driver with `SetExternalDriver` and smoothly rotates the root. The C++ version in VRLab instead calls `SetGrabEnabled(false)`: this both releases the hand and forbids taking the wheel while the door is open. `SetExternalPose` passes the wheel's new world pose with the offset from the door preserved; the manipulation snapshot delivers it to clients. Both external-motion methods are called on the server. This scheme can be used for a handle, a device or another item that after an action must move together with the main assembly.

The geometry of the demo wheel consists of short tangent sections of a smooth ring, a central hub and three spokes from the center to the rim. The generator turns the sections taking the sign of `Roll` in UE into account, so that the rim does not look jagged and the spokes meet exactly at the hub.

`UInteractionDockComponent` is a universal seating support. It accepts a free part with the required Actor Tag, checks distance, orientation and visibility, fixes it to its own transform and automatically removes the attachment after a repeated grab. The support itself can be attached to a component or its socket with standard UE tools. In the demo the piston has `Interaction.EnginePiston`, sits in the engine and moves with it.

## Strike and UI

`UInteractionStrikeComponent` checks the edge speed, the sweep between the previous and current pose, the approach normal and the contact episode. The speed is computed over the time between two poses of the item, not over one server frame: a remote hand moves the item only when a pose packet arrives, so frames without movement are skipped and do not inflate the strike speed. The event does not deal damage by itself: the game decides whether to split a prepared target, play a sound or give haptic feedback. In VR Lab three valid axe strikes open the prepared halves of the log.

The world-space panel is an ordinary two-hand actor with an independent `WidgetComponent`. When a hand holds the frame, it does not simultaneously press a button: action capture belongs to the interactor. The other hand or a separate pointer can press `TEST POINTER`; reset calls a server cleanup of all demo states.

## Launching VR Lab

Step-by-step control of all items, including the bow and arrow, is in [10 — How to grab and use items in the demo](10-Demo-Controls.md). It also describes choosing a palm, finding an item in the emulator and retrying after a failed grab.

1. Open `/InteractionSystem/Demo/VRLab/Maps/VRLab` and press Play.
2. Without an HMD the emulator turns on automatically: WASD/mouse — move and look; holding LMB/RMB — left/right hand; `1/2` chooses the palm to edit; `Alt+mouse` moves it sideways/up, the mouse wheel forward/back; `Z/C` and `R/F` rotate it; `Q/E` perform use/UI. `Tab`/`Backspace` switch between stations, `Home` resets the demo.
3. Quest Touch is already bound in `IMC_VRLab`: Grip — hold, Trigger — use/UI, A — next station, X — previous, B — reset. The pawn has two OpenXR motion controllers and picks HMD mode automatically if `bForceEmulation=false`. A station teleport drops active grips before moving. Continuous thumbstick movement is not implemented in this demo.
4. Check that every physical target has simple collision and only one movement writer. Do not add `ReplicateMovement` in parallel to an actor driven by the manipulation snapshot.

Without a headset, for the rifle or binoculars go with `Tab`/`Backspace` to the needed station, step up so the model is in front of you and aim at the main handle. Hold **RMB** for the right hand: the item stays in the first hand while the button is pressed. Press `1`, bring the left palm to the second point with `Alt+mouse` and the wheel (rotate with `Z/C`, `R/F` if needed), then hold **LMB**. Moving the two palms rotates the item; the rifle and binoculars keep their size. For scaling check the next stations with the engine and the panel. Releasing LMB removes the supporting hand, releasing RMB the last one. The local spheres show the position of both palms.

In your own project enable OpenXR and Enhanced Input, assign `AInteractionVRGameMode` and register `IMC_VRLab` in **Enhanced Input → Default Mapping Contexts** before the XR session starts. The pawn adds the context to the local input subsystem. For Quest 2 set up the Android SDK according to the requirements of your UE 5.8 installation and check the build and comfort on the device itself.

The device mode is determined only on the player's machine after the pawn gets control. `PawnClientRestart` reconnects local input if possession arrived after `BeginPlay`; an owned RPC tells the server whether near VR search or the emulator ray is needed. The absence of an HMD on a dedicated server does not change the VR client's hand mode.

The map, meshes, material instances, profiles and input assets are already part of the plugin. Open them in the Content Browser with **Show Plugin Content** enabled to adapt the example to your game. Gameplay is implemented in the plugin's modules; no extra scripts are needed to run the demonstration.
