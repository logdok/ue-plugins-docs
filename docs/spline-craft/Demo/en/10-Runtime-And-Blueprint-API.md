# Runtime, collision, events, presets, scenarios

*🇬🇧 English | [🇺🇦 Українська](../uk/10-Runtime-And-Blueprint-API.md)*

> ⚠️ **This is the SplineCraft Demo (evaluation) edition.** Every feature works exactly like the full plugin; only *how much* you can build with it is capped, an on-screen watermark is shown, and it does nothing in Shipping builds. Full list: [Demo Limitations](00-Demo-Limitations.md).

## Rebuilding in game

- **`UpdateAfterChangeAnyProperty()`** (`BlueprintCallable`) — call after changing any of the
  structure's properties at runtime. Works **only if Mobility ≠ Static**. It aligns the
  points, rebuilds the geometry, respawns the additional actors and rebinds the hit events.
- **Mobility** (`EComponentMobility`) — mobility of every **generated** component. Affects
  lighting and whether components take part in baking. For a runtime rebuild set `Movable` or
  `Stationary`. A generated component is never left *more static* than **Root Mobility** —
  set that to `Movable` and the meshes follow.
- **Root Mobility** (`EComponentMobility`, default `Static`) — mobility of the actor's own
  spline root. Leave `Static` for baked lighting and the pre-7.x behaviour. Set `Movable` to
  attach a movement component (e.g. *Rotating Movement*) or to move / rotate the whole actor
  at runtime — set **Mobility** to `Movable` too so the generated meshes move with it.
- **SplineComponent** — read-only from Blueprint.

## The Stats section (read-only)

Refreshed after every build, never saved into the level:

| Field | What it shows |
|---|---|
| **Instanced Mesh Components** | how many `InstancedStaticMeshComponent`s the last build produced (one per unique mesh in Line mode). |
| **Total Instances** | how many mesh instances sit across all those components. |
| **Spline Mesh Components** | how many `SplineMeshComponent`s (one per element per span in Curve mode — climbs fast). |
| **Polygon Components** | how many procedural meshes the polygons produced. |

## Collision (`FDemoSCCollisionSettings`)

The **Collision** section on the actor. The settings apply **uniformly to every** generated
component — instanced meshes, spline meshes and polygons alike.

| Parameter | Purpose |
|---|---|
| **Override Collision** | turn collision control on. While off, the legacy behaviour is kept: ISM components stay at the engine default and Spline Mesh components are forced to `Query Only`. |
| **Collision Profile** | the collision profile for every component. |
| **Generate Hit Events** | needed for the `OnSplineCraft…HitEvent` delegates to fire at all. Also needs a profile with physics collision, such as `BlockAll`. |
| **Generate Overlap Events** | generate overlap events. |
| **Polygons Create Collision** | build collision geometry for polygons. Turn it off for purely decorative shapes. |

> Instanced components are keyed by "mesh + materials", so one component is shared by posts /
> sections / tubes / knobs that use the same mesh. That is why the collision settings are
> global to the actor — they cannot be split per element type without splitting the ISM.

## Hit events

`FDemoSplineCraftStaticMeshHitDelegate` (`BlueprintAssignable`), bound in `BeginPlay`:

- **OnSplineCraftSectionHitEvent**
- **OnSplineCraftTubeHitEvent**
- **OnSplineCraftPostHitEvent**
- **OnSplineCraftKnobHitEvent**

Parameters: `Sender`, `HitComponent`, `OtherActor`, `OtherComp`, `NormalImpulse`, `Hit`.

For the events to fire: **Override Collision** on, **Generate Hit Events** on, and a collision
profile with physics blocking (`BlockAll` or similar).

## Presets (`UDemoSplineCraftPreset`)

A style-container asset. The **Preset** field on the actor and two editor buttons:

- **Apply Preset** — load the style onto the actor.
- **Save To Preset** — store the actor's current settings into the asset.

A preset carries: **Mode**, **Visible Last Point Elements**, **Posts**, **Tubes**,
**Sections**, **Knobs**, **Free Knobs**, **Polygons**, **Collision**, **Use Random Seed** /
**Random Seed**, **Mobility**.

A preset **does not touch** what belongs to one particular actor: the spline shape, the
alignment tools, **Spline Close Loop**, **Openings** and **Design Notes**.

Create one: Content Browser → right click → *Miscellaneous → Data Asset* → class
**SplineCraftPreset**. Ready-made samples are in `Content/Samples`.

## Build Scenario — building step by step

The **SplineCraft Demo Build Scenario** asset (`UDemoSplineCraftBuildScenario`) is an ordered, timed
sequence of presets. An actor with a scenario assigned becomes a **director**: it shows none
of its own geometry, and spawns one child SplineCraft Demo actor per step — along its own spline,
with the step's offset. So one preset raised on Z becomes the floors of a building. `Reverse`
is the same sequence taking the structure back down.

### Actor settings (the Build Scenario section)

| Parameter | Purpose |
|---|---|
| **Build Scenario** | reference to the scenario asset. Empty — the actor behaves normally. |
| **Play On Begin Play** | start the scenario automatically when the game begins. |
| **Preview Progress** | *editor only*, 0→1: scrub to see the build with no timers. `0` clears the preview. |

### Methods (Blueprint)

| Method | Effect |
|---|---|
| **Play Build Scenario** (`bReverse`) | start / restart. `false` builds up, `true` takes a finished structure back down. |
| **Stop Build Scenario** | stop and remove every step it has spawned. |
| **Finish Build Scenario Now** | settle at the end state at once. |
| **Is Playing Build Scenario** | whether the scenario is running. |

### Events

- **On Build Scenario Step Shown** (`StepIndex`, `Label`) — a step appeared.
- **On Build Scenario Finished** — the sequence finished.

### The scenario asset

`UDemoSplineCraftBuildScenario`: **Description**, **Start Delay** (s), **Loop** (+ **Loop Restart
Delay**, forward playback only), **Steps**.

Step (`FDemoSCBuildStep`): **Label**, **Preset**, **Delay Before Show** (s), **Offset**
(`FTransform` relative to the director — raise it on Z for floors), **Appear Mode**,
**Appear Duration** (s), **Rise Distance** (cm).

Appear modes (`EDemoSCScenarioAppearMode`): **Instant**, **Rise From Below** (rises from
`Rise Distance` below its place), **Scale Up** (grows from nothing), **Material Progress**
(drives a scalar parameter `SplineCraftBuildProgress` from 0 to 1 on the step's materials —
works only if the material reads that parameter).

The step child actors are `Transient` (never saved into the level) and forced `Movable`. It
is fully additive: without a scenario assigned the actor's behaviour is unchanged.

## See also

- [Performance and Baking](13-Performance-And-Baking.md)
- [Additional Actors](12-Additional-Actors.md)
