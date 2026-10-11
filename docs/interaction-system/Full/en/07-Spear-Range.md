# 07 — Spear throwing

*🇬🇧 English | [🇺🇦 Українська](../uk/07-Spear-Range.md)*

The map `/InteractionSystem/Demo/SpearRange/Maps/SpearRange` is fully part of the plugin. In the Content Browser enable **Show Plugin Content**, open **InteractionSystem Content → Demo → SpearRange → Maps → SpearRange** and press **Play**.

Five spears stand on a rack. When a local player with a free hand approaches a spear, a yellow Nearby dot appears on it. Aim the screen center at the shaft: the dot changes to E and the highlight marks the selected spear. Press **E**, choose a direction and press **F**. While this hand holds a spear, the rest of the spears show no E or Nearby; after the throw they can be selected again. In VR the free second hand works independently. WASD moves, the mouse looks. The throw speed is 28 m/s; for distant targets aim higher, taking gravity into account.

A tip hit on a target, the ground or the fence fixes the spear in the surface. Walk up and press E to pull it out and throw it again. Side hits bounce off naturally. The marks on the field count 5, 10, 20, 30 and 40 meters from the throwing line.

## Structure of the example

- `AInteractionDemoSpear` adds local highlighting, a yellow Nearby dot, an E icon and free-grip alignment to the World's `AInteractionSpearActor`. The proximity radius is set by a non-conflicting overlap sphere around the middle of the spear; the icons are seen only by the local player and disappear while holding. The spear stands vertically in the rack, but after grabbing its +X axis points forward along the player's gaze.
- `AInteractionSpearDemoCharacter` sets up the prompts, the selection distance, the spear's position in the hand and the throw speed. The interaction uses the public `UInteractorComponent`.
- `AInteractionSpearDemoGameMode` selects this character and the shared demo HUD.
- `SM_Spear` is a root static mesh 230 cm long with simple collision. It is hidden and responsible for grabbing and physics. Seven child components draw the shaft, tip, pommel, handle and three brass wrappings without extra collision. The tip direction is +X; the pivot is at the lower end.
- `DA_SpearPhysics` sets a mass of 2 kg, initial rest and alignment to velocity in flight.

The mesh, material instances, data asset and map are stored under `/InteractionSystem/Demo/SpearRange`. The highlight and the E/F images are shared with InteractionLab, also inside the plugin. Core does not depend on Demo; the flight and attachment behavior stays in World.

To change the look open `/InteractionSystem/Shared/Materials`: `M_SurfaceMaster` has the parameters `BaseColor`, `Roughness`, `Metallic`, and `MI_Wood`, `MI_Metal`, `MI_Leather`, `MI_Matte` and `MI_Painted` set the type of finish. The child `MI_*` in SpearRange change only the needed values. For example, `MI_BrushedSteel` inherits metallicity from `MI_Metal`, and `MI_Brass` has its own color. The highlight material has a separate base graph `M_FocusMaster → MI_Glow → InteractionLab/Materials/MI_Focus`. Changing an inherited parameter of a parent instance updates its child variants without editing the maps.

The server drives the grab, throw and hit. To check in the editor you can run two clients through **Play As Client**. How to exclude the demo from the packaged game is described in [06 — Demo content](06-Demo.md).
