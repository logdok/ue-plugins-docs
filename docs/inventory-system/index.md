# Inventory System

<!-- last-synced:start -->
<p style="text-align: right; font-size: .75rem; opacity: .7;"><em>Docs last synced: 2026-08-31 11:33 UTC</em></p>
<!-- last-synced:end -->

A data-driven, fragment-based inventory and equipment system for **Unreal Engine 5.7 and 5.8** with first-class multiplayer support. Items are authored as Data Assets; gameplay code calls a single method and the plugin does the rest — including network replication.

Two documentation trees are available, one per plugin edition, each in English and Ukrainian:

| | English | Українська | Covers |
|---|---|---|---|
| **Full** — the complete plugin | [Guide](Full/en/README.md) | [Посібник](Full/uk/README.md) | Everything: architecture, authoring items, fragments, multiplayer, save/load, the Blueprint API, C++ extension points |
| **Demo** — evaluation build | [Guide](Demo/en/README.md) | [Посібник](Demo/uk/README.md) | The same guide with demo-edition naming (`UISDemoInventoryComponent`, …), plus what the demo specifically limits |

## Which edition is this for?

Both editions expose an identical feature set — every chapter in the Full guide has a matching chapter in the Demo guide, just with `Demo` worked into the type names (`UISInventoryComponent` → `UISDemoInventoryComponent`). The only *behavioral* differences — a five-item-type session budget, an on-screen watermark, no Shipping builds, and an automatic stand-down when the full plugin is installed alongside it — are covered in the Demo guide's [Demo Limitations](Demo/en/00-Demo-Limitations.md) chapter. If you're not sure which one you're using, check your project's `Plugins/` folder: `InventorySystem` is the full edition, `InventorySystemDemo` is the demo. Both editions can be installed in the same project at once, side by side.
