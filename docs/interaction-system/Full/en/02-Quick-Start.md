# 02 — Quick Start

*🇬🇧 English | [🇺🇦 Українська](../uk/02-Quick-Start.md)*

One complete example: a desktop character picks up an item, throws it and opens a door. After every step you can see the result. Step-by-step exercises with explanations are in [14 — Step-by-step tutorials](14-Tutorials.md).

## Steps

1. Enable **Edit → Plugins → InteractionSystem** in a UE 5.8 project and restart the editor.

   **Result:** `InteractorComponent` appears in **Components → Add**.

2. In the character Blueprint enable **Replicates** and add an `InteractorComponent`, attaching it to the camera. Keep `Use Pawn Viewpoint` on: the source will use the controller's current view point.

   **Result:** in the components window the source sits under the camera.

3. Create a Blueprint from `InteractionMasterItem`. Assign `Mesh Component` a static mesh with simple collision that responds with **Block** on the `Visibility` channel. Place the item in front of the camera within 300 cm.

   **Result:** in the game a small dot appears near the item, and `E` appears when you aim at it.

4. In your input mapping bind one action to `Request Grab` and another to `Request Release`. To release without a throw, pass zero vectors; to throw, pass a velocity in cm/s and an angular velocity in degrees/s.

   **Result:** after the first action the item ends up in front of the camera; the second action releases or throws it.

5. For a door or a button implement `Interaction` and call `Request Interaction` instead of `Request Grab`. The ready-made physics actor also has a server event `On Interacted`. A step-by-step example of a door that every player sees is [lesson 2](14-Tutorials.md#lesson-2-a-door-that-opens-for-every-player).

   **Result:** the door reacts to the press.

`E` is only the initial prompt text. Change `Interaction Prompt` or assign a texture to `Focus Icon → Sprite` to match your keys. The plugin does not add keys by itself.

> **In plain words.** Aiming is only "pointing a finger". The item gets into your hand when the server agrees: in single-player that is instant, in multiplayer it takes a server round trip.

## If something does not work

Type `InteractionOverlay` in the game console. A panel appears on the right: whether the source sees an item (**Focus**), the state of the hand (**State**) and what the server answered (**Last answer**). The explanation is in [15 — Debug tools](15-Debug-Tools.md).

## Ready-made example

In the Content Browser enable **Show Plugin Content** and open `/InteractionSystem/Demo/InteractionLab/Maps/InteractionLab`. Move with WASD, look with the mouse, `E` grabs or uses the door, `F` throws. The demo character creates its own Enhanced Input in code and does not change your project's settings.

In a C++ module add `InteractionSystem` to the dependencies; for ready-made items also `InteractionSystemWorld` ([12](12-Cpp-Integration.md#module-dependencies)).

## What next

- Seven step-by-step lessons from the first item to network debugging — [14](14-Tutorials.md).
- Character with Enhanced Input, HUD prompts, doors and VR hands in Blueprint — [11](11-Blueprint-Integration.md).
- The same in C++, a custom grab target and the server-side manipulation functions — [12](12-Cpp-Integration.md).
- How the server validates requests and why an action can be rejected — [13](13-Network-Model.md).
- What to know before integrating — [Limitations and known issues](Limitations.md).
