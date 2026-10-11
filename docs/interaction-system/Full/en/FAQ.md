# FAQ

*🇬🇧 English | [🇺🇦 Українська](../uk/FAQ.md)*

## Selection and focus

### Why is an item not selected?

Open the debug panel with the command `InteractionOverlay` ([15](15-Debug-Tools.md)) and look at the **Focus** row. If it is empty, check: the item's collision blocks `TraceChannel` (`Visibility` by default); the item is closer than `TraceDistance`; the tags pass the source's filter; `CanInteract` returns `true`; the hand is not holding another item. The first blocking object on the ray's path can be a wall or another item.

### Why is there focus on the client, but the action is rejected?

The server rechecks the world at the moment of the request. The reason is reported by `Reason` in the `OnActionResult` event and the **Last answer** row of the debug panel:

- `InvalidTarget` — the target does not have **Replicates**;
- `InvalidPose` — the hand is farther than `MaxOriginDistance` from the pawn or hidden from its eyes;
- `NotReachable` — on the server something is between the hand and the item;
- `Claimed` — the item is in another player's hands;
- `Unavailable` — grabbing is off, there is no free slot or there is no ordinary action.

The full table is in [13](13-Network-Model.md#rejection-reasons).

### How do I show the player why it did not work?

Handle `OnActionResult` and turn `Reason` into text: "Someone is holding this item", "Cannot reach" and so on. A ready table of messages is in [lesson 3](14-Tutorials.md#lesson-3-telling-the-player-why-an-action-failed).

### Why does a hand that holds an item show no prompts for other items?

One `UInteractorComponent` is one hand. While it holds an item or waits for the server's answer, `CanFocusNewTarget()` returns `false` and focus is empty. The second VR hand has its own component and chooses independently.

## Grabbing and motion

### Why does an item jump to the hand when picked up?

The profile has `Alignment=Align`: the first hand aligns the grip point (or the item's root) with the palm. This is how the InteractionLab cubes and the SpearRange spears are set up. To keep an item where it was taken, choose `KeepOffset`.

### How do I set exactly where to hold an item?

Add a `UInteractionGripPointComponent` in the right place (or specify a mesh socket), choose `GripSelection=Authored` or `PreferAuthored` and `Alignment=Align`. In detail — in [08](08-VR-Manipulation.md#sockets-components-and-their-absence).

### Can an item be held with two hands?

Yes, if `MaximumHands=2` and both hands belong to one pawn. `TwoHandPivot`, `bAllowScaling` and the scale limits are set by the profile. Two different players do not hold one item.

### Why does an item in the hand pass through walls?

`Movement=Direct` puts the item exactly in the hand's pose every frame without physical collisions. For tools that must press against obstacles use `Movement=Physics`: the body is driven by a physical drive, and if it is stuck and the hand has moved on by `BreakDistance`, the grip is safely cancelled.

### How do I make a drawer, lid or lever that is part of another actor?

Specify the part in the manipulation's `TargetComponentName` and the `Linear` or `Rotary` constraint. The part stays under its parent and moves with it. Step by step — [lesson 4](14-Tutorials.md#lesson-4-a-drawer-that-slides-out-of-a-cabinet).

### How do I temporarily forbid taking an item, for example during a cutscene?

Call `Set Grab Enabled(false)` on the server: new grips are rejected and the current hands safely release the item. The value is replicated, so the prompt disappears on clients too.

### How do I return items to their places?

`Reset Manipulation` on the server releases the hands and returns an item to its home pose, a handle to zero travel and a part of an actor under its parent. The home is remembered at the start of the game and changed through `Set Home Transform` ([08](08-VR-Manipulation.md#returning-to-the-home-pose)). The `InteractionResetAll` command does this for all items at once.

### Why did an actor stop replicating movement after I added manipulation?

The manipulation component turns off the owner's `ReplicateMovement` itself: motion, scale and attachment are carried by its snapshot. Do not turn `ReplicateMovement` on again. You can move an item from gameplay code: the component passes the change to clients itself.

### How do I attach an item to a shelf or a holster?

Without code — `UInteractionDockComponent`: a released part with the right Actor Tag sits in the support, and a repeated grab frees it. From Blueprint or C++ — `Attach To Surface` of the manipulation on the server ([11](11-Blueprint-Integration.md#server-side-manipulation-functions)).

### How do I make a door or a button in Blueprint?

Implement `InteractableInterface` and the `Interaction` event, and change the visible state through a replicated variable with RepNotify. Step by step — [lesson 2](14-Tutorials.md#lesson-2-a-door-that-opens-for-every-player).

## Network

### Why does an item fall out of the hand by itself in multiplayer?

The server releases an item without a throw if no pose sample arrives from the remote hand for longer than `PoseTimeout`, tracking is lost for longer than `TrackingLossGrace`, the pawn lost its controller or grabbing was turned off. An obstacle in front of the other hand does not take the item away. The reasons are in the [network model](13-Network-Model.md#timeouts-tracking-and-forced-release).

### Does an item in my own hand lag behind the controller?

No: the owner sees the item in their hand at once, both in `Direct` and in `Physics`. Only when the physical body on the server hits an obstacle does the owner smoothly move to the server pose ([08](08-VR-Manipulation.md#physics-mode-in-the-owners-hand)).

### Where do I change the network limits: distance, interval, throw speed?

In **Project Settings → Plugins → Interaction System**. A single `InteractorComponent` can override any value with the `Override` flag next to it ([04](04-Multiplayer-and-VR.md#network-limits)).

### Does the system work with AI?

Yes. On the server an AI pawn is considered locally controlled: its `UInteractorComponent` searches for focus and performs `RequestGrab`, `RequestRelease`, `RequestInteraction` without RPCs. Reduce `FocusUpdateHz` if the AI does not need constant focus.

### Does a thrown item deal damage?

Only if you turned it on: `Apply Impact Damage` in the ready-made physics actor's `Object Data`. Then the damage equals the impact impulse × `Damage Per Impulse`, and the instigator is the player who last held the item ([05](05-Physics.md#impacts-sound-and-damage)).

## Integration

### Is OpenXR required?

The core uses ordinary scene components. OpenXR or another tracking provider is set up by your project.

### Is C++ required?

No. The character, items, doors, reactions to events, the server-side manipulation functions and even a custom grab target (`InteractionBlueprintGrabTargetComponent`) can be done in Blueprint ([11](11-Blueprint-Integration.md)). C++ will be needed if you want to subclass the base grab target directly or change a physics actor's reaction to impacts.

### Can I use only the core?

Yes. Implement `IInteractableInterface` on your actors and add only `InteractionSystem` to the dependencies. The ready-made physics actors and manipulation need World.

### How is the throw velocity computed in VR?

It is passed by your pawn in `RequestRelease`: linear in cm/s and angular in degrees/s in world coordinates. The demo VR pawn smooths the change of palm position between frames. The server clamps the values to `MaxThrowSpeed` and `MaxAngularSpeed`.

### Why does the arrow not nock on the string?

The bow and arrow must be in different hands of one pawn, the bow in one hand only, and the arrow's tail closer than `NockRadius` (18 cm) to the string. Step by step — in [10](10-Demo-Controls.md#bow-arrow-and-shot).

## Presentation

### Why is there no outline?

Assign an overlay material or connect your own post-process to custom depth. The stencil alone does not create a visible outline.

### Can I work in split-screen?

The source logic is separated. The built-in mesh overlay is shared by the process; for separate prompts of each viewport create a presenter on the LocalPlayer through `OnFocusChanged`.

## Version and packaging

### Which engine version is supported?

InteractionSystem 1.0 supports only Unreal Engine 5.8. The current documentation is this directory.

### How do I not ship the demo in my game?

Add `+DirectoriesToNeverCook=(Path="/InteractionSystem/Demo")` to your project's `Config/DefaultGame.ini`. The Demo module's code is not excluded this way; to remove it too, see [06](06-Demo.md#excluding-from-the-packaged-game).

### Do the debug tools end up in the game?

In a Shipping build the `InteractionSystemDebug` module registers nothing: there are no commands and the panel does not open. You can remove it entirely by [the instructions](15-Debug-Tools.md#how-to-remove-the-module).
