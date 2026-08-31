# 10 — Serialization

*🇬🇧 English | [🇺🇦 Українська](../uk/10-Serialization.md)*

The plugin has no save system of its own — deliberately. Every project's is
different, and a plugin that imposes its own slot format usually ends up being
worked around.

Instead, the inventory hands you a **plain struct** that you put into your own
`USaveGame`.

---

## What it looks like

```cpp
UCLASS()
class UMySaveGame : public USaveGame
{
    GENERATED_BODY()

public:
    UPROPERTY(SaveGame)
    FISDemoInventorySaveData Backpack;

    UPROPERTY(SaveGame)
    FISDemoEquipmentSaveData Loadout;
};
```

**Saving:**

```cpp
UMySaveGame* Save = Cast<UMySaveGame>(
    UGameplayStatics::CreateSaveGameObject(UMySaveGame::StaticClass()));

Save->Backpack = InventoryComponent->ExportState();
Save->Loadout  = EquipmentComponent->ExportState();

UGameplayStatics::SaveGameToSlot(Save, TEXT("Slot1"), 0);
```

**Loading:**

```cpp
if (UMySaveGame* Save = Cast<UMySaveGame>(
        UGameplayStatics::LoadGameFromSlot(TEXT("Slot1"), 0)))
{
    InventoryComponent->ImportState(Save->Backpack);
    EquipmentComponent->ImportState(Save->Loadout);
}
```

In Blueprint it's all the same: `Export State` / `Import State` are ordinary
nodes.

---

## What exactly is saved

| Data | How it's stored |
|---|---|
| Item type | `FSoftObjectPath` — a path to the asset, not a pointer |
| Slot number | the exact index, so the layout doesn't shift |
| Stack size | as is |
| Instance state | durability, charges, quality — everything from `StatValues` |
| Slot count | so an upgraded backpack loads upgraded |
| Weight limit | for the same reason |

A path instead of a pointer is what lets a save survive a game restart, a
patch, and even an asset move (Unreal redirectors are honoured).

---

## Item state really is saved

This is not "remember the type and count":

```
Sword at 12/100 durability  →  saved  →  loaded  →  sword at 12/100 durability
```

On load the creation hooks are **not** run — otherwise a rusty sword would come
back from the workshop brand-new. `StatValues` are assigned directly, as
restored facts, not as changes.

---

## When to call `ImportState`

Only after the owning actor is fully initialised. From the constructor is too
early.

In practice: `BeginPlay`, or the moment your save system is ready to apply the
data.

`ImportState` **clears the current contents itself** before restoring — no need
to clear beforehand.

---

## What happens if an item no longer exists

The game updated, the asset was deleted, and the player was carrying it.

Such an item is **skipped with a warning in the log**, and the rest loads
normally. One removed item never costs the player the whole save.

`ImportState` returns `false` to signal that not everything was restored — this
is information, not an error. Whether to show anything to the player is up to
you.

---

## Equipment: two differences

**Items are placed straight into slots**, without going through the inventory.
This is deliberate: restoring equipment shouldn't depend on whether there's room
in the backpack.

**Slots the character no longer has** (the body plan changed between versions)
are skipped with a warning.

The order when loading both components:

```cpp
InventoryComponent->ImportState(Save->Backpack);
EquipmentComponent->ImportState(Save->Loadout);
```

On restore, equipment first clears the current loadout **without returning it to
the inventory** — otherwise the backpack would gain copies of things the save
already accounts for separately.

---

## Multiplayer

Both functions are **server-only**. A call on a client is ignored with a
warning.

This is correct: the inventory is authoritative on the server, so it's restored
there, and clients get the result through ordinary replication.

If your game saves progress separately per player, call `ImportState` on the
server for the appropriate `PlayerState` (or pawn — depending on where you
placed the component).

---

## What is NOT saved

| Not saved | Why, and what to do |
|---|---|
| Items on the ground (`AISDemoItemPickup`) | These are level actors. Save them with your own world-save system, or don't save them at all. |
| Chest state | The same — they're actors. Call `ExportState` on `ContainerInventory` if you need it. |
| Consumable cooldowns | Deliberately ephemeral; after loading the item is available right away. |
| Equipment visual actors | Recreated from the fragment when the slot is restored. |

Saving a chest isn't hard — its inventory is the same:

```cpp
Save->ChestContents = Container->GetContainerInventory()->ExportState();
```

---

## Where to next

- Check what was restored: [13 — Debugging](13-Debugging.md)
- The full list of structs: [12 — API Reference](12-API-Reference.md)
