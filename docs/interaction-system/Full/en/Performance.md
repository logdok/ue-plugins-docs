# Performance

*🇬🇧 English | [🇺🇦 Українська](../uk/Performance.md)*

There are no public FPS, memory or bandwidth figures for a production scene yet. Below is the actual cost of each part according to the 1.0 implementation: how often it updates, what physics queries it makes and what it sends over the network. These are code parameters, not benchmark results; measure your own scene.

## Rates

`FocusUpdateHz` and `PoseReplicationHz` are independent; both default to 30 Hz. The first setting limits the focus search of each hand, the second the sending of one pose packet of the pawn's active hands. The rate of the first active interactor is used for the packet. 20 Hz reduces the number of traces or network RPCs respectively but increases the update latency. Local hand tracking and the solver run every frame.

## Component cost

| Part | Where it runs | Rate | Physics queries | Network |
|---|---|---|---|---|
| `UInteractorComponent` focus | The player's machine | `FocusUpdateHz` per hand | 1 line trace or 1 sweep + 1 line trace; if there is no grabbable target, as many again for `Interact`. Near search: 1 overlap sphere + 1 line trace per candidate | — |
| Pose packet | Remote client → server | `PoseReplicationHz` per pawn | On the server 1 line trace per tracked hand | 1 unreliable RPC with all hands |
| An action request | Client → server | On input | Pose check + 1–2 traces | 1 reliable RPC there and 1 back |
| `UInteractionManipulationComponent` | All machines | Every frame while the item is held, moving or simulated; at rest once per `IdleUpdateInterval` (0.5 s), waking at once on motion | None; in `Physics` mode a `PhysicsHandle` during a grip | `State` is marked changed only on a real change; the position is quantized to 0.01 cm, the rotation is a compressed `FRotator` |
| `AInteractionMasterItem` | Clients and listen server | 10 Hz | Reading the proximity sphere overlap (350 cm) | — |
| `UInteractionDockComponent` | Server | 10 Hz | 1 overlap sphere; 1 line trace per candidate | `Occupant` |
| `UInteractionBowComponent` | Server | Every frame while the bow is held; the arrow search at 20 Hz | 1 overlap sphere on search | `NockedArrow`, `DrawFraction` |
| `UInteractionStrikeComponent` | Server | Every frame while the item is held | Up to 12 × 17 sphere sweeps per frame on a fast swing; usually a few | — |
| `AInteractionSpearActor` | Server | Only in flight | — | Through the manipulation snapshot |

Focus first looks for an available grab target; if there is none, it checks ordinary interaction. Each line trace performs one query, a spherical sweep adds a contact visibility check, and near interaction checks the overlap candidates. Reliable RPCs are used for actions, unreliable ones for periodic poses. A packet with the current poses is also attached to an action request.

## What grows with the number of items

- **Manipulation tick.** The component ticks every frame only when there is work: the item is held, an external driver controls it, the body is simulated and awake, or a client is still moving the pose toward the server's. A resting item checks its state once per 0.5 s and wakes on any transform change or physics body wake-up. Hundreds of items lying around barely occupy the game thread.
- **Replication.** During motion (holding, flight, simulation) the snapshot changes and is sent at the actor's network update rate. Changes smaller than 0.01 cm, 0.01° or 0.001 of scale are not marked. A resting item sends nothing; if push model is on in the build, the server does not even compare its unchanged snapshot (see the tips table below).
- **Master item.** Each item checks the proximity sphere once per 0.1 s. For thousands of items use `AInteractionPhysicsActor` and a centralized HUD, proximity culling or your own presenter. A naive tick of every master item is not claimed to be optimal for thousands of items.
- **CCD.** `AInteractionPhysicsActor` turns on CCD so that thrown items do not fly through thin walls. For numerous small items that are not thrown, CCD can be turned off in a subclass.

## Practical tips

| Situation | What to do |
|---|---|
| Many hands or players | `PoseReplicationHz = 20`; for a desktop player with one interactor 20 is enough |
| Slow targets, large scenes | `FocusUpdateHz = 20`, a narrow `TraceDistance`, a separate `TraceChannel` with only the needed objects |
| Many free items | Reduce the **Net Update Frequency** of item actors and set up relevancy; do not add manipulation to static decorations at all |
| Heavy scenes with physics | `PostThrowBehavior = DisablePhysicsFixedTime` for items that must stop reliably |
| An AI pawn with an interactor | Reduce `FocusUpdateHz`: on the server the AI searches for focus just like a player |
| Edge strikes | A short edge and a larger `EdgeRadius` reduce the number of sweeps; frames in which the item did not move are skipped |
| Hundreds of items on a level | Keep the default `IdleUpdateInterval`; increase it for decorations that are rarely touched |
| Many networked items | Turn on push model: `net.IsPushModelEnabled=1` in `[SystemSettings]` of `DefaultEngine.ini`. Editor builds always support it; for a game and a dedicated server add `bWithPushModel = true` to their `Target.cs` (this requires a unique build environment, that is an engine built from source) |

## How to measure

1. **Unreal Insights:** run the game with `-trace=default,net -NetTrace=1` and look at the time of the `TickComponent` of the manipulation, the interactor and the master item on the game thread.
2. **`stat game`** and **`stat physics`** in the console show the overall tick time and physics queries.
3. **Network Insights** in the same trace (the `Net` channel) shows the size and rate of `ServerUpdatePoses` and the replication of `State`.
4. For the network in PIE add latency and loss: `Net PktLag=100`, `Net PktLoss=5`.

The debug panel ([15](15-Debug-Tools.md)) costs nothing while closed: the answer log only records `OnActionResult` events. An open panel updates its numbers four times a second. In Shipping the module registers nothing. The current limits of the model are described on the [Limitations and known issues](Limitations.md) page.
