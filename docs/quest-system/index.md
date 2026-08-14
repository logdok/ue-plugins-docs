# QuestSystem

An event-driven, data-driven quest system for **Unreal Engine 5.7–5.8** with first-class multiplayer support and zero-config integration. Designers author quests as Data Assets; gameplay code reports a handful of events; the plugin does the rest — including the network replication.

Two documentation trees are available, one per plugin edition, each in English and Ukrainian:

| | English | Українська | Covers |
|---|---|---|---|
| **Full** — the complete plugin, no limits | [Guide](Full/en/README.md) | [Посібник](Full/uk/README.md) | Everything: architecture, authoring, multiplayer, save/load, the Blueprint API, C++ extension points |
| **Demo** — evaluation build | [Guide](Demo/en/README.md) | [Посібник](Demo/uk/README.md) | The same guide with demo-edition naming (`UDemoQuestData`, …), plus what the demo specifically limits |

## Which edition is this for?

Both editions expose an identical feature set — every chapter in the Full guide has a matching chapter in the Demo guide, just with `Demo`-prefixed type names. The only *behavioral* differences — a session quest budget, an on-screen watermark, no Shipping builds — are covered in the Demo guide's "Demo Limitations" chapter. If you're not sure which one you're using, check your project's `Plugins/` folder: `QuestSystem` is the full edition, `QuestSystemDemo` is the demo. Both editions can be installed in the same project at once, side by side.
