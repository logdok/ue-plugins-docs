# Interaction System

<!-- last-synced:start -->
<p style="text-align: right; font-size: .75rem; opacity: .7;"><em>Docs last synced: 2026-10-11 00:32 UTC</em></p>
<!-- last-synced:end -->

One shared interaction model for **Unreal Engine 5.8** on desktop, mobile and VR: ray-based selection with
local focus feedback, **server-authoritative grabbing**, one- and two-hand manipulation with profiles,
physics items, a bow and arrow, rotary and linear handles, and a built-in debug panel. Works from
Blueprint and from C++, in single-player and multiplayer.

| | English | Українська |
|---|---|---|
| **Full** — the complete plugin | [Guide](Full/en/README.md) | [Посібник](Full/uk/README.md) |

## What it covers

A source on the pawn (a camera or a VR hand) finds a target with a ray or a sphere and shows focus at
once. The server validates every request, decides who holds what through a claim ledger, and explains
every rejection with a reason. Items can be held by two hands of one player, scaled, thrown, attached
to a surface or driven along a rotary or linear constraint. The documentation doubles as a tutorial:
it has a glossary, seven step-by-step lessons and a debug-tools chapter.

## Where to start

New to interaction systems? Read the [Quick Start](Full/en/02-Quick-Start.md), then the
[step-by-step tutorials](Full/en/14-Tutorials.md); unfamiliar words are in the
[glossary](Full/en/Glossary.md) (Ukrainian: [Швидкий старт](Full/uk/02-Quick-Start.md),
[покрокові уроки](Full/uk/14-Tutorials.md), [словник](Full/uk/Glossary.md)).

When something does not work, open the debug panel with the console command `InteractionOverlay`
([Debug tools](Full/en/15-Debug-Tools.md)): it shows focus, hand state and the server's answer.
