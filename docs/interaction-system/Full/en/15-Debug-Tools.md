# 15 — Debug tools

*🇬🇧 English | [🇺🇦 Українська](../uk/15-Debug-Tools.md)*

The most frequent question during integration is "why is the item not taken?" The `InteractionSystemDebug` module answers it without reading code: console commands print the state of the system, and an on-screen panel shows focus, hands, held items, the claim ledger and the server's answers in real time.

> **In plain words.** This is the system's "dashboard". Instead of guessing why an action failed, you see exactly what the server answered and which rule fired.

The tools work in the editor, in PIE and in Development builds. **They are absent from Shipping builds:** the commands are not registered and the panel does not open, even if a Blueprint calls the corresponding node.

## How to open the panel

| Way | What to do |
|---|---|
| Console | Press `~` and type `InteractionOverlay`. Repeating the command closes the panel |
| Blueprint | Call `Toggle Interaction Debug Overlay` with your `Player Controller`, for example on a debug key |
| State check | `Is Interaction Debug Overlay Open` returns whether the panel is open |

On a **PIE client** first type `EnableCheats`: Unreal creates the cheat manager only in single-player and on the server, so without this command the client answers "Command not recognized".

The panel does not pause the game: the character is still controlled, and a cursor appears so you can press the panel's buttons.

## What the panel shows

| Section | Contents |
|---|---|
| Header | The role of the machine: `STANDALONE`, `LISTEN SERVER`, `DEDICATED SERVER` or `CLIENT`. A client sees its own hands and what the server has already sent |
| Buttons | **Refresh** — rebuild the panel; **Release held** — release everything the local pawn holds, without a throw; **Reset objects** — return all items to their home places (on a client it sends `InteractionResetAll` to the server); **Close** |
| Session | The level, the player's pawn and the world time |
| Local sources | Each `InteractorComponent` of the local pawn: the hand (`LEFT`/`RIGHT`/`ANY`), state, focus, held item, the server's last answer and the source's effective network limits |
| Held objects | For each held item: the movement mode (`DIRECT`, `PHYSICS`, `SCRIPTED`), the number of hands, the handle travel or "Free", attachment and the update rate ("Every frame" or "Idle") |
| Claims (server) | Who holds or reserves each item. The ledger lives on the server, so on a client the section explains how to get the data with a command |
| Recent answers | The last eight server answers to this machine's requests: time, hand, action, target and verdict. Hover over a red rejection to read the explanation |

### Source states

| State | Meaning |
|---|---|
| `Ready` | The hand is free, searches for focus and can take |
| `Waiting for the server` | A grab request was sent and there is no answer yet |
| `Holding` | The hand holds an item |
| `Tracking lost` | The hand sends no reliable pose (`SetTrackingValid(false)`) |
| `Inactive` | The component is disabled |

## Console commands

| Command | Arguments | What it does | Where it runs |
|---|---|---|---|
| `InteractionOverlay` | — | Open or close the panel | Locally |
| `InteractionStatus` | — | Print each local source: state, focus, held item, the server's last answer | Locally |
| `InteractionLog` | `[Count]`, 10 by default | Print the last server answers, newest first; an explanation under each rejection | Locally |
| `InteractionClaims` | — | Print the claim ledger: item → pawn and hands, linked reservations | On the server; a client forwards the command through `ServerExec` |
| `InteractionRelease` | — | Release everything the local pawn holds, as a cancellation: nothing is thrown | Locally; the component informs the server itself |
| `InteractionResetAll` | — | Return all items with manipulation to their home places | On the server; a client forwards the command |

The command answers appear in the game console and in the log with the category `LogInteractionSystemDebug`. The answer of a server command returns to the client's console.

The answer log is kept from the start of the game even when the panel is closed. An answer to a request arrives only where the request was sent from: a client sees the answers to its own actions, a listen-server host to its own.

## Recipe: "why is it not taken?"

1. Open the panel and aim the hand at the item.
2. **Focus** is empty — the client does not even consider the item a target. Check the collision (blocks `Visibility`), the distance (`Trace Distance`), the filter tags, `Can Interact` and whether the hand is holding another item.
3. **Focus** shows the item, but after an attempt **Last answer** is red — the server rejected the request. The reason is in the same row; the explanation is in the tooltip:
   - `InvalidPose` — the server did not believe the hand position: check `Max Origin Distance` and whether geometry hides the hand from the pawn's eyes;
   - `NotReachable` — on the server something stands between the hand and the item or the item is already elsewhere;
   - `Claimed` — another player holds the item; `InteractionClaims` shows who;
   - `Unavailable` — the item refused: grabbing is off (`Set Grab Enabled`), there is no free hand, the profile is invalid;
   - `InvalidTarget` — the item is not replicated (in a networked game).
4. The state is stuck at `Waiting for the server` — the request did not arrive or the component is disabled on the server. Check that the pawn replicates and the component is active on the server.
5. The item is taken but behaves strangely — look at **Held objects**: the movement mode, the number of hands, the constraint, attachment.

## How to remove the module

The module adds nothing to a Shipping build, so it is usually kept. To remove it from a project completely, delete the `InteractionSystemDebug` entry from the `.uplugin` and the `Source/InteractionSystemDebug` directory. No other module depends on it; the plugin's tests skip the debug checks if the module is absent.

## What next

- The meaning of each rejection reason and the order of checks — [13 — Network model](13-Network-Model.md#rejection-reasons).
- A step-by-step exercise with the panel — [lesson 7](14-Tutorials.md#lesson-7-testing-in-multiplayer-and-finding-causes).
