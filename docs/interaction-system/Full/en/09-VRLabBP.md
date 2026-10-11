# 09 — VRLabBP: the Blueprint demo

*🇬🇧 English | [🇺🇦 Українська](../uk/09-VRLabBP.md)*

`/InteractionSystem/Demo/VRLabBP/Maps/VRLabBP` is a separate map in the InteractionSystem 1.0 plugin for Unreal Engine 5.8. In the Content Browser enable **Show Plugin Content**, open the map and press **Play**. Quest 2 uses OpenXR and Touch controllers; without a headset a two-hand emulator starts with [the same keys as in VRLab](08-VR-Manipulation.md#launching-vr-lab). `Tab` and `Backspace` switch between the seven stations. The map does not change the existing InteractionLab, SpearRange and VRLab.

The scene has a light stone floor, cobalt walls, blue and turquoise accents, gold pointers and a red wheel. The bright palette with its parent instances is in `/InteractionSystem/Shared/Materials`. The VRLab map uses the same instances; changing a color updates both scenes without repainting the meshes.

Before playing open the [step-by-step instructions for grabbing and using items](10-Demo-Controls.md): they explain how to place the palm spheres, hold two mouse buttons, take the bow and arrow, draw the string and shoot.

| Station | Blueprint actors | What to check |
|---|---|---|
| 01. Interaction and throwing | `BP_PickupCube`, `BP_UseButton`, `BP_ThrowableSpear` | Take the cube, press the red button, throw the spear at the target. The button handles `Interaction` in its own Event Graph. |
| 02. Two hands | `BP_TwoHandRifle`, `BP_TwoHandBinoculars` | The rifle uses `Grip_Main`/`Grip_Support` on the mesh; the binoculars use two component anchors without a socket. |
| 03. Assembly inspection | `BP_ScalableEngine`, `BP_RemovablePiston` | Two hands scale and rotate the engine; `PistonSeat` accepts a released piston with the tag `Interaction.EnginePiston`. |
| 04. Spatial panel | `BP_SpatialPanel` | Resize the frame with the hands, press `TEST POINTER` with a separate pointer. `WorldControls` is attached to the mesh and scales with the frame. |
| 05. Bow | `BP_Bow`, `BP_Arrow` | Hold the bow and arrow in different hands, nock the arrow, draw and release. `UpperString` and `LowerString` are visible even without an arrow. The bow's Event Tick updates both sections by the draw of `BowInteraction`; `OnShot → Launch` launches the arrow. |
| 06. Bunker | `BP_BunkerWheel`, `BP_BunkerDoor` | Take the rim of the red three-spoke wheel and make two full turns. The wheel's Event Graph checks the travel. The door smoothly rotates 100° around the side hinge, and the wheel moves with the leaf. |
| 07. Strike | `BP_TwoHandAxe`, `BP_ChoppingLog` | Take the axe with two hands and hit the log with the blade. `OnStrike` on the server calls the event `SplitNow`, which sets the replicated `bSplit`; the function `OnRep_bSplit` hides the intact log and shows the two prepared halves on all machines. |

The **RESET ALL DEMONSTRATIONS** button on the panel, `Home` in the emulator or B on Quest Touch first release active grips on the server. Then `BP_VRLabPawn.OnLabResetRequested` runs a server travel back to `VRLabBP`: the door, log, button and all items return to their initial state.

Open `BP_Bow`, `BP_BunkerWheel`, `BP_BunkerDoor`, `BP_TwoHandAxe`, `BP_ChoppingLog`, `BP_UseButton` and `BP_VRLabPawn` to view their Event Graphs.

`BP_ChoppingLog` is a model of how to make a state visible to everyone in Blueprint. The strike event runs only on the server, so you cannot change meshes directly in it: other players would not see it. Instead the variable `bSplit` has **Replication = RepNotify**, and the whole visual change lives in the function `OnRep_bSplit`. The server calls it itself after **Set w/ Notify**, clients when they receive the new value, including a player who joined after the strike.

> **In plain words.** The server does not draw the split log for everyone — it only announces "the log is split". Every machine, hearing the announcement, hides the intact log and shows the halves itself.

The other items show composition from `UInteractionManipulationComponent`, `UInteractionGripPointComponent`, `UInteractionDockComponent`, profiles and ordinary Blueprint properties. Grabbing, network ownership, physics and the two-hand solver remain the implementation of the reusable Core/World modules. `BP_VRLabPawn` and `BP_VRLabGameMode` are Blueprint assets; for XR input, the desktop emulator and station transitions the pawn inherits the ready-made `AInteractionVRPawn`. The spatial widget uses `UInteractionVRPanelWidget` from the Demo module. So the examples of item behavior can be studied and changed in Blueprint without rewriting the low-level XR input.

All 16 Blueprint assets, the map and the materials are part of the plugin. Open a Blueprint in the editor to view or change the graphs and component settings; no extra scripts are needed to run. For Quest 2 check your own Android build and the behavior on the headset.
