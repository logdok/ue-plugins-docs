# InteractionSystem

*🇬🇧 English | [🇺🇦 Українська](../uk/README.md)*

One shared interaction model for desktop and VR: ray-based selection, server-authoritative grabbing and local feedback.

Version [1.0](Release-Notes.md), Unreal Engine 5.8.

## Quick start

1. Enable **Edit → Plugins → InteractionSystem** in a UE 5.8 project and restart the editor.
2. Add an `InteractorComponent` to a replicated pawn and attach it to the camera. For desktop keep `Use Pawn Viewpoint` on; for VR add one component per hand and turn this option off.
3. Create a Blueprint from `InteractionMasterItem`, assign a mesh with collision that blocks `Visibility`, and place it within 300 cm of the player.
4. Bind one key to `Request Grab` and another to `Request Release` with velocities in cm/s and degrees/s. For doors use `Request Interaction` and implement `Interaction`.
5. Press **Play**: a marker appears near the item, an `E` prompt appears when you aim at it, and after you press the key the item ends up in your hand.
6. If something does not work, type `InteractionOverlay` in the console: the panel shows focus, hand state and the server's answer ([15](15-Debug-Tools.md)).

A ready-made example is the map `/InteractionSystem/Demo/InteractionLab/Maps/InteractionLab`: WASD and mouse to move, `E` to grab or open the door, `F` to throw. More detail in [02 — Quick Start](02-Quick-Start.md).

## How to read this documentation

| You are… | Start with |
|---|---|
| Building item interaction for the first time | [01 — Introduction](01-Introduction.md) → [02 — Quick Start](02-Quick-Start.md) → [14 — Step-by-step tutorials](14-Tutorials.md); unfamiliar words are in the [glossary](Glossary.md) |
| Working in Blueprint only | [11 — Blueprint integration](11-Blueprint-Integration.md) |
| Writing C++ | [12 — C++ integration](12-Cpp-Integration.md) and the [API reference](API-Reference.md) |
| Building multiplayer or VR | [04 — Multiplayer and VR](04-Multiplayer-and-VR.md), [08 — VR manipulation](08-VR-Manipulation.md), [13 — Network model](13-Network-Model.md) |
| Looking for the cause of a problem | [15 — Debug tools](15-Debug-Tools.md) and the [FAQ](FAQ.md) |

## Contents

- [Release notes](Release-Notes.md)
- [01 — Introduction](01-Introduction.md)
- [02 — Quick Start](02-Quick-Start.md)
- [03 — Interaction and focus](03-Interaction-and-Focus.md)
- [04 — Multiplayer and VR](04-Multiplayer-and-VR.md)
- [05 — Physics items](05-Physics.md)
- [06 — Demo content](06-Demo.md)
- [07 — Spear throwing](07-Spear-Range.md)
- [08 — VR manipulation and profiles](08-VR-Manipulation.md)
- [09 — VRLabBP: the Blueprint demo](09-VRLabBP.md)
- [10 — How to grab and use items in the demo](10-Demo-Controls.md)
- [11 — Blueprint integration](11-Blueprint-Integration.md)
- [12 — C++ integration](12-Cpp-Integration.md)
- [13 — Network model](13-Network-Model.md)
- [14 — Step-by-step tutorials](14-Tutorials.md)
- [15 — Debug tools](15-Debug-Tools.md)

## Reference

- [Glossary](Glossary.md)
- [API reference](API-Reference.md)
- [Architecture](Architecture.md)
- [Performance](Performance.md)
- [Limitations and known issues](Limitations.md)
- [FAQ](FAQ.md)
