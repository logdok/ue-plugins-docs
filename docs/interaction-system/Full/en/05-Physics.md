# 05 — Physics items

*🇬🇧 English | [🇺🇦 Українська](../uk/05-Physics.md)*

`AInteractionPhysicsActor` has a mesh root and a `UInteractionManipulationComponent`. The component passes a single snapshot of motion, scale, hands and attachment; the actor's standard `ReplicateMovement` is turned off so that two mechanisms do not overwrite the transform. `ObjectData` references a `UInteractionPhysicsDataAsset`.

- `MassInKg` sets the mass; the mesh must have suitable simple collision.
- `bStartWithPhysics` turns simulation on from the start.
- `SleepWhenSettled` keeps the natural Chaos sleep: a hit from another body can wake the item.
- `DisablePhysicsFixedTime` stops the simulation completely after `PhysicsDisableDelay`; such an item will not wake from an ordinary physical hit, but it can be grabbed again.
- `HitSound` is played through a multicast after an authoritative impact.

The server tells the actor about pickup and release through the interface: `OnGrab(Interactor)` is called when the first hand is accepted, `OnRelease(Interactor, Reason, LinearVelocity, AngularVelocity)` when the last one lets go. In `OnGrab` the ready-made actor remembers the player as `Instigator`, and in `OnRelease` it starts the `DisablePhysicsFixedTime` timer. When overriding these functions in a subclass, call the parent implementation.

The constructor turns on the `PhysicsActor` collision profile for the mesh root, **Simulation Generates Hit Events**, wake events and CCD (`bUseCCD`), so that a fast item does not fly through thin walls. CCD has a cost: for a large number of small stationary items it can be turned off in a subclass.

## Impacts: sound and damage

While an item is free (nobody holds it), the server handles every hit event of `MeshComponent`. All thresholds are set by `ObjectData`:

| Setting | Default | What it does |
|---|---|---|
| `HitSound` | — | Impact sound; played by an unreliable multicast at the impact point on all clients |
| `HitSoundMinimumImpulse` | 150 | Weaker impacts make no sound: touches and resting are quiet |
| `HitSoundCooldown` | 0.15 s | The shortest gap between two sounds of one item, so that sliding does not repeat the sound every frame |
| `bApplyImpactDamage` | off | Whether to call `UGameplayStatics::ApplyDamage` for the actor the item hit |
| `DamagePerImpulse` | 0.005 | Damage per unit of normal impulse |
| `MinimumDamageImpulse` | 20 | Weaker impacts deal no damage |
| `DamageTypeClass` | `UDamageType` | The damage type passed to `ApplyDamage` |

> **In plain words.** Impulse is the "strength" of a blow: it grows with the item's mass and speed. A thrown brick has a large impulse; one placed on a table has almost zero. The thresholds cut off minor touches so that sound and damage come only from real impacts.

Damage is off by default: turn on `bApplyImpactDamage` if thrown items should wound or break things. The damage instigator is the controller of the pawn that **last held** the item: `OnGrab` assigns it as `Instigator`. For your own rules override `OnComponentHitInternal` in a C++ subclass.

With `Movement=Direct` a grabbed item keeps `QueryOnly` so the second hand can select it. With `Movement=Physics` one server PhysicsHandle drives the body while keeping physical collisions. After release `QueryAndPhysics` is used; `bSimulateOnRelease` and `bAllowThrow` determine simulation and throw. Linear velocity is in cm/s, angular in degrees/s. All physical actor callbacks have an authority guard; do not drive a copy of the client body by hand. Profiles are described in [08 — VR manipulation](08-VR-Manipulation.md).

`AInteractionSpearActor` uses the +X axis as the tip direction. The tip lies at `(Bounds.Max.X, 0, 0)` of the local mesh; the pivot may be shifted along the shaft. `GetSpearTipLocation()` returns its world position. A spear can align to its velocity and attach to a surface at an angle of up to 60°, a speed from `MinimumEmbedSpeed` (200 cm/s) and contact in the front section `TipContactLength` (40 local cm). A side or slow hit leaves an ordinary physical reaction.

In flight the spear ticks only on the server. If `ObjectData.bAlignToVelocity` is on and the speed exceeds 100 cm/s, the actor smoothly turns along the velocity vector (`RInterpTo` with `AlignmentStiffness`; 20 in SpearRange), and the angular velocity is zeroed. Flight ends when the body falls asleep, the simulation is off or the spear is taken again. `Launch(Velocity)` starts the same flight without a hand, for example from an arrow released by a bow.

`PenetrationDepth` sets the depth the tip sinks in world centimeters from the contact point. After a hit `IsEmbedded()` returns true, the simulation stops and the spear moves together with the surface. It can be grabbed and thrown again. For network attachment to a moving target its actor and component must be replicable. A ready-made example is [SpearRange](07-Spear-Range.md).

`UInteractionDestructibleComponent` is optional destruction from impacts. Add it to a replicated actor with a primitive root and hit events (the ready-made `AInteractionPhysicsActor` already turns them on). The server reduces health, and when it runs out, all machines show the effect and the actor is destroyed. This is a simple gameplay coefficient, not a physical model of strength.

| Property, event or method | Meaning |
|---|---|
| `Health` | Strength reserve, 50 by default; set in Class Defaults. `GetHealth()` returns the remainder on the server |
| `DamagePerImpulse` | Damage per unit of impact impulse, 0.01 by default |
| `MinimumImpactDamage` | An impact must deal more than this damage (5 by default) or it is not counted |
| `DestroyEffect` | Optional effect at the destruction point: a Niagara or Cascade system |
| `OnImpactDestroyed(Location)` | Fires on the server and on every client before destruction: sound, debris, a custom effect |
| `ApplyImpactDamage(Amount, Location)` | Server damage from your logic: an explosion, a sword strike and so on |

For example, a crate with `Health = 30` and the default coefficients does not notice impacts with an impulse up to 500 (damage up to 5), while an impact with impulse 1500 takes 15 health: two such impacts are enough. A step-by-step exercise is [lesson 6](14-Tutorials.md#lesson-6-a-crate-that-breaks-under-impacts).

## How to choose a mode

| You need | Setting |
|---|---|
| An item lies on a rack and does not fall until first picked up | `bStartWithPhysics=false` in `ObjectData` |
| An item falls and rolls at once | `bStartWithPhysics=true` |
| A thrown item should "fall asleep" and wake on impacts | `PostThrowBehavior=SleepWhenSettled` |
| A thrown item must reliably stop after N seconds | `PostThrowBehavior=DisablePhysicsFixedTime`, `PhysicsDisableDelay=N` |
| An item in the hand must not pass through walls | a profile with `Movement=Physics` ([08](08-VR-Manipulation.md)) |
| Thrown items should wound or break things | `bApplyImpactDamage=true` in `ObjectData`, and if needed `UInteractionDestructibleComponent` on the target |
| A model for inspection that stays in the air after release | `bSimulateOnRelease=false`, `bAllowThrow=false` |
