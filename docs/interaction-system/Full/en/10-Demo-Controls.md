# 10 — How to grab and use items in the demo

*🇬🇧 English | [🇺🇦 Українська](../uk/10-Demo-Controls.md)*

These instructions are for InteractionSystem 1.0 on Unreal Engine 5.8. In the Content Browser enable **Show Plugin Content**. For two hands open `/InteractionSystem/Demo/VRLab/Maps/VRLab` or `/InteractionSystem/Demo/VRLabBP/Maps/VRLabBP` and press **Play**. The hand controls are the same on both maps. If the keys do not respond, first click in the game window so that it receives input.

## Hands without a VR headset

The spheres show the palm positions. The keys `1` and `2` choose the palm you move; they do not grab an item and do not change the mouse button assignments.

| Key or button | Action |
|---|---|
| `WASD`, mouse | Player movement and look |
| `1` / `2` | Choose the left / right palm for moving and rotating |
| Left `Alt` + mouse | Move the chosen palm sideways and up/down |
| Wheel forward / back | Push the chosen palm away from you / pull it toward you |
| `Z` / `C` | Turn the chosen palm left / right |
| `R` / `F` | Turn the chosen palm up / down |
| Hold LMB / RMB | Grab and hold an item with the left / right hand |
| Release LMB / RMB | Release the item with the corresponding hand |
| `Q` / `E` | Use an item or press UI with the left / right hand |
| `Tab` / `Backspace` | Next / previous station; current grips are released |
| `Home` | Reset the demonstration |

**Important:** in the emulator the item is chosen by a short search forward from the palm, in the direction of its rotation. The sphere is not an automatic grab button on touch. Place it slightly in front of the item's surface, on your side, and point the palm at the item. Do not sink the sphere into the handle or pass it through a table or stand: the obstacle check can reject the grab.

For items with defined grip points the palm must be near the right point. In the demo the radius of these points is 20 cm by default; merely hitting another part of the item does not yet allow taking it. After aiming, press and hold a mouse button. If the attempt failed, **release the button, adjust the palm and press again**: the search is performed at the moment of the press, not repeated all the time while holding.

## The cube and the spear on VRLabBP

1. Choose a free palm with `1` or `2` and bring it in front of the cube or the spear shaft.
2. Hold the corresponding button: LMB for the left hand, RMB for the right.
3. Move the palm to move the item; rotate it with `Z/C` and `R/F`.
4. To throw, move the palm forward and release the button during the motion. Releasing a stationary palm only releases the item. The spear can embed in a surface at sufficient speed and a tip hit.

The red button nearby is an example of an ordinary action: point a free palm at it and press `Q` or `E`. You do not need to take it in your hand.

## The rifle and binoculars

For the rifle start with the handle by the trigger; the front handle is intended for the second hand. The binoculars can be started from either side.

1. Press `2`, bring the right palm to the rifle's main handle or one side of the binoculars and hold RMB.
2. While holding RMB, press `1` and move the left palm to the rifle's front handle or the opposite side of the binoculars.
3. Press and hold LMB. Now the item is held with two hands.
4. Move the palms to rotate the item. The size of the rifle and binoculars stays constant.
5. Release one hand's button to leave the item in the other; release both to free it.

These models demonstrate grip and manipulation. Firing the rifle and optical zoom of the binoculars are not implemented in these examples.

## The engine and the removable piston

1. Take the engine body with one hand, then add the second from the other side. Hold LMB and RMB.
2. Choose one palm with `1/2` and move it away from the other: the engine grows. Bring the hands together to shrink it. The scale is limited.
3. Change the palms' positions to rotate the engine and inspect it from different sides.
4. To work with the piston, free one hand, bring it to the piston at the top of the engine and press the corresponding grab button again.
5. Take out the piston. To return it, bring it to the seat at the top of the engine, align it and release. If the position and orientation fit, the docking support fixes the part.

One hand cannot hold the engine and the piston at once. The piston is a separate item.

## The spatial panel

1. Bring one palm to a side frame and grab it.
2. With the other palm grab the opposite side frame.
3. Move the hands apart to enlarge the panel; bring them together to shrink it. Moving the pair of palms also rotates the panel.
4. To press `TEST POINTER` free one hand. Point it at the button and press `Q` for the left or `E` for the right hand.

A hand that holds the frame does not press UI at the same time. The **RESET ALL DEMONSTRATIONS** button resets the whole map.

## Bow, arrow and shot

The order given uses the left hand for the bow and the right for the arrow. The hands can be swapped, together with the grab and use buttons.

1. Press `1`. With `Alt + mouse` and the wheel place the left sphere in front of **the middle of the wooden bow handle**, not near the string or the end of a limb. Press and hold LMB.
2. Without releasing LMB press `2`. Bring the right sphere to **the arrow tail near the fletching**, slightly in front of its surface. Press and hold RMB.
3. Lift the arrow off the stand. Both buttons must stay pressed: the left hand holds the bow, the right the arrow.
4. Bring **the arrow tail to the middle of the string**. The tail must be brought near the string, not the tip or the middle of the shaft. The initial nocking radius is 18 cm. The arrow automatically takes the string and aligns with the bow's direction.
5. Holding the bow still, pull the right palm back **along the arrow, toward its tail**. If the bow points forward from the player, scroll the wheel back to pull the right palm toward you. The string must form an angle and follow the hand.
6. Release **RMB** to shoot. Keep holding LMB so the bow stays in the left hand. Too small a draw produces no shot; the arrow stays nocked on the string.
7. To take off a nocked arrow without a shot, press `E` with the arrow's hand. For the left hand this is `Q`.
8. After the shot you can take the next arrow with the free hand or pick up the previous one, including pulling it out after it hit a surface.

**If the arrow does not nock:** check that the bow and arrow are already grabbed by different hands and that the tail is near the middle of the string. Do not release the arrow's button before nocking. While drawing move the hand along the arrow's axis: a significant offset sideways or up/down cancels the nocking. Releasing the bow itself also cancels the link with the arrow.

## The bunker door wheel

1. Bring a free palm to the rim of the red wheel and grab it with the corresponding mouse button. Start near the authored point on the side of the rim.
2. Hold the button and move the palm **in a circle around the center of the wheel** with `Alt + mouse`. It is the motion of the palm around the axis that turns the wheel; rotating the palm in place with `Z/C` does not unscrew it.
3. If the motion runs into the initial limit, move the palm the other way. Make small smooth steps.
4. You can release the wheel and re-grip it near the grip point in the new position. The turns made are preserved.
5. After two full turns the door smoothly opens, rotating around the side hinge. The wheel stays on the leaf and moves with it; the wheel's active grip is released. You do not need to take the door leaf itself.

## The axe and the log

1. With the first hand take the axe by the end of the handle; bring the second closer to the middle of the handle and add a supporting grip.
2. Hold both buttons. Move and rotate the hands to bring the blade to the log.
3. Swing and drive **the blade**, not the handle, through the log with enough speed. A slow touch does not count as a strike.
4. After a strike move the blade away from the log before the next swing: continuous contact does not produce a series of separate strikes.

On VRLab the log splits after three valid strikes; on VRLabBP the Blueprint example shows the splitting after a valid strike event. Press `Home` to repeat.

## If an item is not taken

- Free the needed hand: LMB controls only the left, RMB only the right, regardless of `1/2`.
- Place the sphere in front of the surface, not inside the item or behind the stand. For an authored grip bring it to the handle or the side point, not to an arbitrary part of the model.
- Check the palm direction: after `Z/C` or `R/F` its search may point away from the item.
- After a failed attempt release the button, adjust the position and press again.
- If the state is unclear, press `Home` and try again from the initial arrangement.

## Quest 2

Bring the controller to the corresponding grip point and hold **Grip**. VR uses near search by the palm. Releasing Grip frees the item; the speed of the hand's motion is passed to the throw if the profile allows it. For two hands hold Grip on both controllers. **Trigger** uses an item or presses UI with the free hand. The bow and arrow are taken with different controllers: bring the arrow's tail to the string, then the second hand draws it and releases Grip for the shot. A goes to the next station, X to the previous, B resets the map.

## InteractionLab and SpearRange without a headset

These two maps use different controls: aim your gaze at an item and press **E** to take it; press **F** to throw it. On VRLab and VRLabBP the F key rotates the chosen palm, and the item is released with the mouse button. Do not carry the E/F controls from the cubes or the range over to the VR maps.
