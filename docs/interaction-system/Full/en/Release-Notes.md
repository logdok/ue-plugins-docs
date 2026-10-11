# Release notes

*🇬🇧 English | [🇺🇦 Українська](../uk/Release-Notes.md)*

A short note on every release: what's new, what's fixed and what to watch for when upgrading. The newest release is at the top.

## 1.0

Unreal Engine **5.8**. The first release of the plugin. The previous prototype has a different API; there is no compatibility layer, so move your integration following the [quick start](02-Quick-Start.md).

**What's inside:** [ray, sphere sweep, focus and tags](03-Interaction-and-Focus.md); [server requests, the claim ledger and two VR hands](04-Multiplayer-and-VR.md); [physics items, the spear and destruction](05-Physics.md); [VR manipulation with profiles](08-VR-Manipulation.md): two hands of one player, socket/component/free anchors, scaling and rotation, physical holding, a bow and arrow, a wheel, part seating and strike events.

**Network:**

- The server validates the pose of each hand separately: an obstacle in front of a free hand no longer makes the server drop an item in the other hand, and a locally deactivated first hand does not stop pose updates.
- `OnActionResult` reports the rejection reason `EInteractionRejectReason`: `Cooldown`, `InvalidTarget`, `InvalidPose`, `NotReachable`, `HandBusy`, `Claimed`, `Unavailable` ([13](13-Network-Model.md#rejection-reasons)).
- The network limits are set in **Project Settings → Plugins → Interaction System** and overridden on a single `InteractorComponent` ([04](04-Multiplayer-and-VR.md#network-limits)).
- `bGrabEnabled` and `bCanBeGrabbed` are replicated; `SetGrabEnabled` turns grabbing off and safely releases the current hands.
- A physical item in the owner's hand is predicted and does not lag by a server round trip.
- The manipulation snapshot is quantized and supports push model; resting items tick once per 0.5 s and wake on motion ([Performance](Performance.md)).

**Manipulation:**

- A named part of an actor (`TargetComponentName`) — a drawer, a lid, a lever — stays under its parent, moves with it and returns to its place after a reset.
- `ResetManipulation` returns an item to the home pose from the start of the game; `SetHomeTransform` changes it.
- `BreakDistance` is counted from the body's closest approach, so a physical item taken from afar is not released at once.
- `AttachToSurface`, `SetFreeSimulation`, `IsAttached`, `SyncTransform` and the claim ledger queries are available in Blueprint.

**Targets:**

- `UInteractionBlueprintGrabTargetComponent` lets you describe a custom grab target in Blueprint ([11](11-Blueprint-Integration.md#custom-grab-target-in-blueprint)).
- The interface events `OnGrab(Interactor)` and `OnRelease(Interactor, Reason, LinearVelocity, AngularVelocity)` come from the core for any target; Blueprint can implement them but not call them.
- An actor with only a target component gets focus only when it can be taken right now.

**Physics and impacts:**

- Impact damage of a physics actor is off by default and configured in `ObjectData`; the impact sound has a threshold and a pause between repeats ([05](05-Physics.md#impacts-sound-and-damage)).
- `UInteractionDestructibleComponent` accepts Niagara or Cascade effects, has configurable thresholds and an `OnImpactDestroyed` event.
- The strike speed of `UInteractionStrikeComponent` is computed over the time between the item's poses.

**Tools:** the `InteractionSystemDebug` module with an on-screen panel and the commands `InteractionOverlay`, `InteractionStatus`, `InteractionLog`, `InteractionClaims`, `InteractionRelease`, `InteractionResetAll` ([15](15-Debug-Tools.md)). In Shipping builds the module registers nothing.

**Demo:** InteractionLab and [SpearRange](07-Spear-Range.md) with five spears, targets, distance marks and E/F buttons; the VRLab map with seven stations and a desktop emulator; the [Blueprint map VRLabBP](09-VRLabBP.md) with the same mechanics in Event Graphs. The desktop character uses Enhanced Input created in code. The split log in VRLabBP is replicated to all players. VRLab and VRLabBP use a shared palette from `/InteractionSystem/Shared/Materials`. All the content is inside the plugin; the Demo and Debug modules can be removed, and the tests skip their checks.

**Documentation:** [step-by-step tutorials](14-Tutorials.md), a [glossary](Glossary.md), integration in [Blueprint](11-Blueprint-Integration.md) and [C++](12-Cpp-Integration.md), the [network model](13-Network-Model.md), the [API reference](API-Reference.md) and a [limitations](Limitations.md) page.

There is no separate CHANGELOG.md — changes are recorded on this page.
