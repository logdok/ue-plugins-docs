# 14 — Step-by-step tutorials

*🇬🇧 English | [🇺🇦 Українська](../uk/14-Tutorials.md)*

Seven short lessons for those who are building item interaction for the first time. Each lesson has a goal, steps and a **"What you should get"** item: if the result differs, there are hints at the end of the lesson. The lessons go from simple to complex, but each can be done on its own. Terms that may be unfamiliar are explained in the [glossary](Glossary.md).

For all lessons:

- the plugin is enabled in **Edit → Plugins → InteractionSystem** and the editor is restarted;
- in the **Content Browser → Settings** **Show Plugin Content** is enabled, so you can see the plugin's classes and demo;
- the project is on Unreal Engine 5.8; the **First Person** or **Third Person** template will do.

## Lesson 1. A first item you can pick up

**Goal:** the player aims their gaze at a crate, presses `E` — the crate ends up in front of them; `F` — throws the crate.

**Time:** 10 minutes.

### Steps

1. Open your character's Blueprint. In **Components → Add** add an `InteractorComponent` and drag it onto the camera. In **Details** keep `Use Pawn Viewpoint` on and set `Grab Offset` to `(70, 0, -15)`.

   > **In plain words.** `InteractorComponent` is "a hand with a laser pointer". The pointer (the ray) shows what you want to take, and `Grab Offset` is where this hand holds what it took: 70 cm in front of the camera and slightly lower.

2. Create **Content Browser → Add → Blueprint Class → All Classes → InteractionMasterItem** and name it `BP_Crate`. In `Mesh Component` assign `Cube` and set the scale to `0.4`.
3. Drag `BP_Crate` onto the level about 2 m in front of the player's start.
4. Create the Input Actions `IA_Use` and `IA_Throw` (type **Digital (bool)**) and the Input Mapping Context `IMC_Interaction`: `E → IA_Use`, `F → IA_Throw`. Add the context to the player the same way the template adds its own (the **Add Mapping Context** node in **Event Receive Controller Changed** or in **Begin Play**).
5. In the character's Event Graph:
   - **IA_Use → Started** → `Request Grab` (the target is your `InteractorComponent`);
   - **IA_Throw → Started** → `Request Release`: `Linear Velocity` = **Get Forward Vector** of the camera × `900`, `Angular Velocity` = `(0, 120, 0)`.
6. Press **Play**.

### What you should get

- When you approach the crate, a small dot appears above it: an item is nearby.
- When you aim the screen center at the crate, the dot changes to `E` and the crate is highlighted.
- After `E` the crate ends up in front of the camera and moves with you.
- After `F` the crate flies forward and falls; it can be taken again.

### If it did not work

| What you see | What to check |
|---|---|
| Neither a dot nor `E` | The crate is closer than 300 cm (`Trace Distance`); the mesh collision blocks the **Visibility** channel |
| `E` is there but the crate is not taken | Whether `Request Grab` is called (add a **Print String**); whether the input context is active |
| The crate is taken but does not fly | Whether you pass a non-zero `Linear Velocity` to `Request Release` |

For any unclear case open the debug panel with the command `InteractionOverlay` ([15](15-Debug-Tools.md)): it shows focus, the state of the hand and the server's answer.

## Lesson 2. A door that opens for every player

**Goal:** a door opens when `E` is pressed, and all players see it open.

### Steps

1. Create a Blueprint Actor `BP_Door`, and in **Class Defaults** enable **Replicates**.
2. Add a Static Mesh `Leaf` (the door leaf) with collision and **Mobility = Movable**.
3. **Class Settings → Interfaces → Add** → `InteractableInterface`.
4. Add a variable `bOpen` (Boolean) and in **Details → Replication** choose **RepNotify**. The editor will create the function `OnRep_bOpen`. In it: `Leaf → Set Relative Rotation` with `Yaw = Select(bOpen, 90, 0)`.
5. Implement **Event Interaction**: `Set bOpen = NOT bOpen`.
6. Override the function `Can Interact`: return `true` if `Action == Interact`.
7. In the character from lesson 1 add a second action or change `IA_Use`: if **Get Focused Actor** cannot be grabbed (it is not an `InteractionPhysicsActor` or has `Can Be Grabbed = false`), call `Request Interaction` instead of `Request Grab`. The ready logic of such a choice is described in [11](11-Blueprint-Integration.md#input-with-enhanced-input).
8. Check in the network: **Play → Net Mode → Play As Client**, **Number of Players = 2**.

> **In plain words.** `Interaction` runs on the server, as if the referee wrote down "the door is open". RepNotify is the mailing of this news: every client receives the new `bOpen` and turns the leaf in its own copy of the world.

### What you should get

One player presses `E` near the door — in **both** windows the door turns 90°. Another `E` closes it for both.

### If it did not work

- The door opens in only one window: the leaf is turned directly in `Interaction` and not through `bOpen` with RepNotify.
- The door does not react on the client: `BP_Door` does not have **Replicates** enabled. The server rejects actions with non-replicated actors (reason `InvalidTarget`).

## Lesson 3. Telling the player why an action failed

**Goal:** when the server rejects an action, the player sees a clear message.

### Steps

1. Select the character's `InteractorComponent` and in **Details → Events** click **+** next to `On Action Result`.
2. The event gives `Target Actor`, `Action`, `Accepted` and `Reason`. If `Accepted = false`, connect **Switch on EInteractionRejectReason** and output text (**Print String** or your widget):

| `Reason` | What happened | Example message |
|---|---|---|
| `Cooldown` | Pressed more often than `Min Action Interval` allows | "Not so fast" |
| `InvalidTarget` | The target disappeared or is not replicated | "This item is unavailable" |
| `InvalidPose` | The server did not believe the hand position: far from the body, behind a wall, stale | "Your hand is too far" |
| `NotReachable` | The server's ray did not reach the target | "Cannot reach" |
| `HandBusy` | The hand already holds something | "Your hand is busy" |
| `Claimed` | Another player holds the target | "Someone is holding this item" |
| `Unavailable` | The target refused: grabbing is off, no room, no action | "Not possible right now" |

3. Check: start two players (**Play As Client**, 2 players), take the crate with the first player and try to take the same crate with the second.

### What you should get

The second player sees "Someone is holding this item": the server answered `Claimed`.

> **In plain words.** `On Action Result` is the referee's answer to your application. It does not only say "no", it also explains which rule fired.

## Lesson 4. A drawer that slides out of a cabinet

**Goal:** a drawer in a cabinet slides out after the hand, but no more than 40 cm, and moves together with the cabinet.

### Steps

1. Create a Blueprint Actor `BP_Cabinet` with **Replicates**. The root is the Static Mesh of the body.
2. Add a child Static Mesh `Drawer`: **Mobility = Movable**, collision blocks **Visibility**. Do not enable **Component Replicates**: the manipulation carries the drawer's motion.
3. Add an `InteractionManipulationComponent` and in its **Details**:
   - `Target Component Name = Drawer` — move only the drawer, not the whole cabinet;
   - **Inline Settings**: `Maximum Hands = 1`, `Constraint = Linear`, `Constraint Axis` — the sliding direction in the drawer's axes (for example `(1, 0, 0)` if its front panel looks along +X), `Minimum Value = 0`, `Maximum Value = 40`, `Simulate On Release` off.
4. Optionally choose `InteractionManipulationComponent` → **Events → On Handle Changed**: when `Fraction ≥ 0.99`, play the sound of a fully extended drawer.
5. **Play**: take the drawer (`E` from lesson 1) and step back.

### What you should get

- The drawer travels only along its axis and stops after 40 cm, however far you step back.
- After release the drawer stays extended and remains part of the cabinet.
- If the cabinet moves (for example you move it in Blueprint), the drawer moves with it and still slides out relative to the cabinet.
- A call of `Reset Manipulation` on the server closes the drawer.

> **In plain words.** `Constraint = Linear` is rails for the drawer. `Minimum Value` and `Maximum Value` are the stops at the ends of the rails in centimeters.

### If it did not work

- The whole cabinet is taken: `Target Component Name` is not set or has an error in the name.
- The drawer is not taken: the profile has `Maximum Hands = 2` (a constraint needs one hand) or `Movement = Physics` (a constraint needs `Direct`). The reason is shown by `Validate Setup` and the debug panel.

## Lesson 5. A Blueprint lantern: a custom grab target without C++

**Goal:** a lantern that is taken into the hand, switched on with the use button and stays where it was put. All the rules are in Blueprint.

### Steps

1. Create **Blueprint Class → All Classes → InteractionBlueprintGrabTargetComponent** and name it `BP_LanternGrab`. This is a component in which you yourself describe what happens on a grab.
2. Open `BP_LanternGrab`, and in **Functions → Override** add:
   - **Begin Grip** (`In Hand`): `Get Owner → Attach Actor To Component` with `Parent = In Hand`, rules `Snap to Target`; return `true`;
   - **End Grip** (`In Hand`, `In Reason`, velocities): `Get Owner → Detach From Actor` with `Keep World`;
   - **Use** (`In Hand`): toggle the owner's light (see step 4) and return `true`.
3. Create a Blueprint Actor `BP_Lantern`: enable **Replicates** and **Replicate Movement** (so the attachment to the hand reaches clients). The root is a Static Mesh with collision that blocks **Visibility**; a child **Point Light**.
4. In `BP_Lantern` add a variable `bLightOn` with **RepNotify**; in `OnRep_bLightOn` switch the light's visibility. In `BP_LanternGrab → Use` call `Set bLightOn = NOT bLightOn` on the owner (**Cast To BP_Lantern**).
5. Add `BP_LanternGrab` to the components of `BP_Lantern` and put the lantern on the level.
6. In the character: if the hand already holds an item (`Get Held Actor` is valid), pressing `E` should call `Request Interaction` — this is "use the item in the hand".

### What you should get

- `E` on the lantern — it ends up in the hand and moves with you; the second player sees the lantern in your hand.
- Another `E` — the light turns on or off for everyone.
- `F` — the lantern stays hanging where you released it (there is no physics: your `End Grip` decided so).

> **In plain words.** The plugin keeps the role of the referee: who may take, whether anyone else is holding, whether the hand is too far. And what to do with the lantern after permission is decided by your Blueprint.

### Good to know

- **Can Begin Grip** is called both on the client (for the prompt) and on the server. Read only replicated state in it, change nothing.
- **Begin Grip**, **End Grip** and **Use** run only on the server.
- `In Reason = Released` means an intentional release; other reasons are cancellations, and the item should not be thrown then.
- `Set Grab Enabled(false)` on the server forbids new grabs and releases the current hands.

## Lesson 6. A crate that breaks under impacts

**Goal:** a crate withstands several strong impacts and bursts apart with an effect that everyone sees.

### Steps

1. Create a Blueprint from `InteractionPhysicsActor` — `BP_FragileCrate`; assign a mesh.
2. Create a Data Asset from `InteractionPhysicsDataAsset` with `Start With Physics = true` and assign it in `Object Data`.
3. Add an `InteractionDestructibleComponent`. In **Class Defaults** set `Health = 30`, `Destroy Effect` — any Niagara system (or Cascade), `Damage Per Impulse = 0.01`, `Minimum Impact Damage = 5`.
4. Optionally: **Events → On Impact Destroyed** → play a sound at the point `Location`. The event fires on the server and on every client.
5. **Play**: take another crate (from lesson 1) and throw it at `BP_FragileCrate` several times, or throw `BP_FragileCrate` itself at a wall.

### What you should get

Light touches do nothing. After several strong impacts the crate disappears with a destruction effect in all windows.

> **In plain words.** The damage from an impact = the strength of the impact × `Damage Per Impulse`. Impacts weaker than `Minimum Impact Damage` are not counted, so a crate that just lies there or slides slowly does not wear out.

### If it did not work

- The crate does not break: reduce `Health` or increase `Damage Per Impulse`; make sure the crate simulates physics.
- You want to break the crate with a sword strike or an explosion: call `Apply Impact Damage(Amount, Location)` on the server from your logic.

## Lesson 7. Testing in multiplayer and finding causes

**Goal:** learn to see what the server sees and to find the cause when an item "is not taken".

### Steps

1. **Play → Net Mode → Play As Client**, **Number of Players = 2**, press **Play**.
2. In the client window open the console (`~`) and type `EnableCheats` — on a PIE client the debug commands are unavailable without it.
3. Type `InteractionOverlay`. A panel appears on the right.
4. Aim the hand at an item and try to take it. Look at the **State**, **Focus** and **Last answer** rows in the **Local sources** card.
5. Type `InteractionClaims`: the server prints to the console who holds what.
6. Add network latency: `Net PktLag=150`. Take an item again and see how the **Waiting for the server** state lasts longer.

### What you should get

- The panel shows the role `CLIENT`, your sources, focus and the server's answer to each attempt.
- During the latency the item in your hand still moves at once: the motion in the owner's hand is predicted, and the server only confirms it.
- The messages in **Recent answers** explain every rejection; hover over one to read the explanation.

The panel and all commands are covered in detail in [15 — Debug tools](15-Debug-Tools.md).

## What next

- The full description of Blueprint integration — [11](11-Blueprint-Integration.md).
- VR hands, two-hand items and profiles — [08](08-VR-Manipulation.md).
- How the server makes decisions — [13](13-Network-Model.md).
