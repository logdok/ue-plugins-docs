# 01 — Introduction

*🇬🇧 English | [🇺🇦 Українська](../uk/01-Introduction.md)*

InteractionSystem **1.0** is an interaction system for **Unreal Engine 5.8**: the player aims their gaze or hand at an item, sees a prompt, picks the item up, uses it and throws it. The same model works for desktop and VR, in single-player and multiplayer.

## What problem it solves

"Pick up an item" looks like one ray and one attachment. In practice harder questions appear quickly:

- how to show a prompt only for what can really be picked up, and not highlight an item behind a wall;
- what to do when two players grab one item at the same moment;
- how to stop a cheater from "reaching" an item across the map;
- how to hold an item with two VR hands, scale a panel, turn a wheel or nock an arrow;
- how to keep an item in a hand from jittering for other players.

Without a ready-made system every project solves this again: a ray in the character's Tick, an actor attached to a hand, a home-made "busy" flag and a series of network fixes. InteractionSystem collects these solutions into tested components.

## How it is organized: four roles

| Role | Who | What it does |
|---|---|---|
| **Source** | `UInteractorComponent` on a pawn: a camera or a VR hand | Searches for an item with a ray or sphere, shows focus, sends requests to the server |
| **Target** | An actor with `IInteractableInterface` or a grab target component | Says whether interaction is allowed and reacts: opens, switches on, is picked up |
| **Judge** | The server and the `UInteractionClaimSubsystem` ledger | Checks every request and decides who holds what |
| **Motion executor** | `UInteractionManipulationComponent` on an item | Moves a held item with one or two hands and sends its state to clients |

> **In plain words.** Imagine a museum with a cloakroom. You point at a coat (source and focus), the attendant checks your ticket (judge and ledger), and only then is the coat handed over (motion executor). Anyone can point, but only the attendant hands out the coat.

The client shows local focus immediately, so prompts appear without delay. The server repeats the checks and makes the decision: allow the action, reject it with a reason, or release the item. The plugin does not impose a specific input mapping, VR SDK or inventory system.

## Main features

- A thin ray or a spherical sweep, near-hand search for VR, filters by Gameplay Tags and Actor Tags.
- Prompts and highlighting for each hand separately; the free second VR hand works independently.
- Server-side validation of every request with an explanation of the rejection (`Claimed`, `NotReachable`, `InvalidPose` and so on).
- An item can be held by two hands of one player: rotation, scaling, grip points on sockets and components.
- Physical holding (the item stops at walls), throwing with a speed limit, a spear that embeds in surfaces.
- Rotary and linear handles — wheels, drawers — including as parts of larger actors.
- Bow and arrow, docking places for parts, edge strikes, destruction from impacts.
- A grab target with its rules in Blueprint, without C++.
- Network limits in **Project Settings → Plugins → Interaction System** with per-component overrides.
- A debug panel and console commands.
- Four demo maps: InteractionLab, SpearRange, VRLab and the Blueprint version VRLabBP.

## Where to start

- Newcomers — [02 — Quick Start](02-Quick-Start.md), then [14 — Step-by-step tutorials](14-Tutorials.md). Unfamiliar terms are explained in the [glossary](Glossary.md).
- Experienced developers — [Architecture](Architecture.md) and [13 — Network model](13-Network-Model.md).
