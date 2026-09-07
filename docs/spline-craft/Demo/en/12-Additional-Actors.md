# Additional Actors

*🇬🇧 English | [🇺🇦 Українська](../uk/12-Additional-Actors.md)*

> ⚠️ **This is the SplineCraft Demo (evaluation) edition.** Every feature works exactly like the full plugin; only *how much* you can build with it is capped, an on-screen watermark is shown, and it does nothing in Shipping builds. Full list: [Demo Limitations](00-Demo-Limitations.md).

Any knob can display an **actor** instead of a mesh — when **Displayed Object = `Actor`**.
This brings lamps with logic, flags with physics, triggers, particle systems — anything that
is a full actor — into the structure. The actors appear **in game only**; in the editor they
are not visible.

## Parameters (`FDemoSCAdditionalActor`)

| Parameter | Purpose |
|---|---|
| **Actor Class** | the actor class to spawn (`TSubclassOf<AActor>`). |
| **Scale** | an additional scale (`FVector`), multiplied on top of the spawn transform. |
| **Spawn Transform** | the computed spawn transform — read-only. |

The **Shift** and **Rotation** values come from the settings of the knob the actor is attached
to.

## Spawn and lifecycle

- The actors are spawned in **`BeginPlay`** (the `SpawnAdditionalActorsIfNeeded` method),
  after the geometry is built.
- Each spawned actor is **attached** to the SplineCraft Demo actor and given a tag with the owning
  actor's name (`GetFName()`).
- On a rebuild (`CoreClean`) all previously spawned additional actors are destroyed by that
  tag and spawned again.
- At runtime the respawn happens inside `UpdateAfterChangeAnyProperty()` (provided
  **Mobility ≠ Static**).

## Where the `Actor` mode is available

- **Posts / Knobs** — `Main Static Mesh Configuration → Displayed Object = Actor`.
- **Knobs on sections and tubes** (`FDemoSCElementKnob`) — `Displayed Object = Actor`.
- **Free Knobs** — likewise.

## See also

- [Knobs and Free Knobs](05-Knobs-And-Free-Knobs.md)
- [Runtime, collision, events, presets, scenarios](10-Runtime-And-Blueprint-API.md)
