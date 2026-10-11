# Glossary

*🇬🇧 English | [🇺🇦 Українська](../uk/Glossary.md)*

A page for those who are new to interaction, Unreal networking or VR. Each term is explained twice: briefly, as it works in InteractionSystem, and **in plain words** — through an everyday analogy. Terms are in alphabetical order of their English names, because that is how they appear in the editor and in code.

## Authority (the server)

The machine whose version of the world is considered correct. In a networked game this is the server; in a single-player game it is your computer. All InteractionSystem decisions about who holds what are made by the authority.

> **In plain words.** The referee at a match. Players may see a situation differently, but only the referee writes down the score.

## Claim

An entry in the server ledger `UInteractionClaimSubsystem`: "this item is held by this hand of this player". While a claim exists, another player cannot take the item.

> **In plain words.** A cloakroom ticket. While the ticket is in your hand, nobody will give your coat to someone else.

## Constraint

The `Constraint=Rotary` or `Linear` mode in a manipulation profile: the item does not fly freely after the hand but rotates around an axis or slides along it. This is how the bunker wheel, handles and drawers are made.

> **In plain words.** Rails for a little cart: you can push it, but it will only travel along the rails.

## Dedicated server and listen server

A dedicated server is a separate process without a player, only a server. A listen server is a setup where one of the players is also the server (the "host"). For the host of a listen server actions run without network calls.

## Focus

The target the interaction source is "looking at" right now: it is highlighted and has a prompt above it. Focus is computed only on the player's machine and gives no right to act: the server checks everything again.

> **In plain words.** You pointed a finger at a product in a shop window. That is not a purchase yet: the cashier will check that the product is in stock.

## Grab target

A component that allows an actor to be picked up by a hand: `UInteractionGrabTargetComponent` and its subclasses. The ready-made variant is `UInteractionManipulationComponent`; the variant with rules in Blueprint is `UInteractionBlueprintGrabTargetComponent`.

## Grip point

`UInteractionGripPointComponent` — a place on an item that is the right place to hold it: a rifle's handle, a wheel's rim. Without points the item is taken wherever the hand touches it.

## Home transform

The place `ResetManipulation` returns an item to. It is remembered at the start of the game and changed through `SetHomeTransform`.

> **In plain words.** A book's place on the shelf: after reading it is put back exactly there.

## Interactor (interaction source)

`UInteractorComponent` on a pawn: a desktop player's camera or one VR hand. The source looks for focus, sends requests to the server and holds the item.

> **In plain words.** A hand with a laser pointer: the pointer shows what you want to take, and the hand takes it.

## Manipulation

`UInteractionManipulationComponent` — the component that moves a grabbed item: after one hand, two hands, with scaling, physically or along a constraint. It is the only one that changes the item's pose.

## Pose and pose packet

A pose is the position and rotation of a source. A pose packet is several poses of all of a player's hands that the client regularly sends to the server. Each hand has its own sample number, and the server validates each hand separately.

> **In plain words.** A photo of your hands that you send to the referee every moment. A blurry photo of one hand does not cancel the photo of the other.

## Prediction

The client shows the result immediately without waiting for the server. InteractionSystem predicts the motion of an item in the owner's hand: for both `Direct` and `Physics`. Grabbing is not predicted: the item appears in the hand after the server's answer.

## Push model

An Unreal replication mode in which the server compares a variable only when code has marked it as changed. The manipulation snapshot supports push model; how to enable it in your build is on the [Performance](Performance.md#practical-tips) page.

## Reject reason

`EInteractionRejectReason`: why the server rejected a request — `Cooldown`, `InvalidTarget`, `InvalidPose`, `NotReachable`, `HandBusy`, `Claimed` or `Unavailable`. It arrives in the `OnActionResult` event and is shown in the debug panel.

## Replication

Automatic copying of variables and actors from the server to clients. What is not replicated, clients do not learn about.

> **In plain words.** A news mailing from the server. If a piece of news did not make it into the mailing, players do not know about it.

## RepNotify

A replicated variable with an `OnRep_...` function that the client calls as soon as it receives a new value. A convenient way to show a change to everyone: a door, a split log, a lamp turned on.

## RPC (remote procedure call)

A function that a client calls on the server (`Server`) or the server calls on clients (`Client`, `NetMulticast`). All InteractionSystem requests are RPCs of your pawn's `UInteractorComponent`.

## Snapshot

The `State` structure of the manipulation component: item pose, hands, attachment, physics, handle travel. The server updates the snapshot, and clients smoothly move the item toward it.

## Trace and sweep

A trace finds the first object along a thin line. A sweep does the same, but with a "thick" sphere. The source uses them to find focus, and the server to validate a request.

## Tracking

Whether the system knows where the hand is right now. For a VR controller tracking is lost when the headset camera cannot see it. Report it through `SetTrackingValid`.

## What next

- How these parts work together — [01 — Introduction](01-Introduction.md).
- First practical steps — [14 — Step-by-step tutorials](14-Tutorials.md).
