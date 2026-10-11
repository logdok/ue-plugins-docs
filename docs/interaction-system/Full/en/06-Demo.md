# 06 — Demo content

*🇬🇧 English | [🇺🇦 Українська](../uk/06-Demo.md)*

Open **Content Browser → Settings → Show Plugin Content → InteractionSystem Content → Demo → InteractionLab → Maps → InteractionLab** and press **Play**.

Three items on the stand demonstrate proximity markers, precise ray selection, the E texture and an emissive overlay. The door on the right demonstrates an ordinary server `Interaction` action without grabbing. The HUD hints at the available action and the grab state; while an item is held, an F button labeled `THROW` appears to the right of the explanation panel.

When one hand holds a cube, the other cubes show no E or Nearby for that hand: it is already busy. Press F to release the cube and pick the next one. In VR the free second hand can choose separately.

The cubes use `Alignment=Align`: after pressing E their root moves to the palm at the configured distance from the camera (70 cm forward and 15 cm down). This also turns the item to match the gaze direction. If you need to keep the initial distance and angle at the moment of grabbing, change `Manipulation → InlineSettings → Alignment` to `KeepOffset`.

| Input | Action |
|---|---|
| WASD | Movement |
| Mouse | Look |
| E | Grab an item or open/close the door |
| F | Throw the grabbed item |

The demo character uses Enhanced Input: the actions and the `IMC_DemoDesktop` context are created in code when the player gets control and removed when they lose it. So the demo works in any project and does not change its input settings. For your own game do the same or create Input Action assets ([11](11-Blueprint-Integration.md#input-with-enhanced-input)).

For a network check choose **Play → Advanced Settings → Multiplayer Options**, two players and **Play As Client**. Each client has its own focus; only one player can grab a single item. To see what is happening, type `EnableCheats` on the client, then `InteractionOverlay` ([15](15-Debug-Tools.md)).

The `InteractionSystemDemo` module contains the character, HUD, door, game mode, the [spear-throwing example](07-Spear-Range.md) and [VR Lab](08-VR-Manipulation.md). It uses the public Core/World API, Engine, Enhanced Input, XRBase/HeadMountedDisplay and UMG/Slate; Core and World do not depend on Demo. The maps, meshes and textures are under `/InteractionSystem/Demo`, and the shared materials under `/InteractionSystem/Shared/Materials`. The original buttons are the textures `T_InteractE`, `T_NearbyGreen`, `T_NearbyYellow` and `T_ReleaseF` in `/InteractionSystem/Demo/InteractionLab/UI`: E/F have an opaque background, and both proximity markers are just a colored dot on a transparent background. InteractionLab uses the yellow dot with a black outline; green remains available as an alternative texture.

The base graphs, parent instances and the bright palette of both VR maps are in `/InteractionSystem/Shared/Materials`. VRLab and VRLabBP refer to the same `MI_Cobalt`, `MI_Cyan`, `MI_Gold`, `MI_HoneyWood` and other color variants. InteractionLab and SpearRange have their own child instances for their separate examples. To change the shared VR palette, edit the corresponding instance in the Content Browser.

## VR Lab

The map `/InteractionSystem/Demo/VRLab/Maps/VRLab` demonstrates seven stations: an axe and a log, a rifle, binoculars, a scalable engine with a removable piston, a spatial UI panel, a bow with arrows and a bunker door with a red three-spoke wheel. Holding and behavior are set by profiles and components; the specific demo actors add visual feedback.

The finished map can be edited in Unreal Editor: geometry, collision, sockets, profiles and input assets are saved as ordinary plugin content. Gameplay runs in the plugin's C++ modules. The Quest Touch and emulator controls are given in [08 — VR manipulation](08-VR-Manipulation.md).

The separate map [VRLabBP](09-VRLabBP.md) contains seven bright stations and Blueprint actors with Event Graphs for the button, the bow, the bunker wheel and the log splitting. The split log is replicated, so every player sees it, including those who joined later.

## Excluding from the packaged game

In a project that uses the plugin add to `Config/DefaultGame.ini`:

```ini
[/Script/UnrealEd.ProjectPackagingSettings]
+DirectoriesToNeverCook=(Path="/InteractionSystem/Demo")
```

If you added the demo maps to `MapsToCook` in your project, remove those entries when excluding Demo and choose your own start map. To package any demo map, add it to `MapsToCook`.

This setting excludes the content but not the C++ Runtime module. To remove the code too in a source distribution, delete the `InteractionSystemDemo` entry from the `.uplugin`, the `Source/InteractionSystemDemo` directory and `Content/Demo`. Core and World do not depend on it.

The debug module does not need `InteractionSystemDemo` either: `InteractionSystemDebug` is a separate leaf module that registers nothing in a Shipping build. How to remove it as well is in [15](15-Debug-Tools.md#how-to-remove-the-module). The plugin's tests skip the demo and debug checks when the corresponding modules are absent.
