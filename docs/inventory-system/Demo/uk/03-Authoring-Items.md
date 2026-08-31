# 03 — Створення предметів

*[🇬🇧 English](../en/03-Authoring-Items.md) | 🇺🇦 Українська*

Як створити тип предмета, повний довідник його полів і готові рецепти типових
предметів.

---

## Створення ассету

**Content Browser → правий клік → Miscellaneous → Data Asset → ISItemDefinition.**

Розташування — будь-де у вашому контенті. Плагін шукає предмети через Asset Registry
за класом, тож структура тек ваша, і переміщення ассетів нічого не ламає.

> **Одне застереження щодо назви.** Ім'я ассета — це стабільна ідентичність предмета:
> саме воно потрапляє у збереження й у консольні команди. Перейменування ассета після
> релізу зробить недійсними посилання в чужих save-файлах. `DisplayName` натомість
> можна змінювати будь-коли — на нього ніхто не посилається.

---

## Довідник полів

### Подання

| Поле | Опис |
|---|---|
| `DisplayName` | Назва, яку бачить гравець. Локалізується як звичайний `FText`. |
| `Description` | Довший текст для підказки чи панелі деталей. |
| `Icon` | Іконка для UI. **М'яке посилання** — див. нижче. |
| `PickupMesh` | Меш для предмета, що лежить на землі. Також м'яке посилання. |

**Чому іконка — м'яке посилання.** Володіння предметом не тягне текстуру в пам'ять.
Ваш UI сам вирішує, коли її завантажити (Blueprint: `Async Load Asset`; C++:
`StreamableManager`). Завдяки цьому тисяча типів предметів не затягує тисячу текстур
у пам'ять виділеного сервера, якому вони взагалі не потрібні.

### Класифікація

| Поле | Опис |
|---|---|
| `ItemTags` | Чим предмет **є**: `Item.Type.Weapon.Melee`, `Item.Rarity.Rare`, `Item.Property.Tradeable`. |

Ці теги читають фільтри інвентаря, обмеження слотів і запити. Зазвичай предмет має
один тег типу, один тег рідкості й скільки завгодно тегів властивостей.

Збіг **ієрархічний**: предмет із тегом `Item.Type.Weapon.Melee` відповідає й на
запит про `Item.Type.Weapon`.

### Здатності

| Поле | Опис |
|---|---|
| `Fragments` | Що предмет **уміє**. Тут і відбувається дизайн предметів. |

Порядок має значення лише для `OnUsed` — фрагменти виконуються згори вниз.

Докладно про кожен фрагмент — [04 — Фрагменти](04-Fragments.md).

### Похідні значення (лише для читання)

Ці функції рахуються з фрагментів, і їх зручно викликати з UI:

| Функція | Що повертає |
|---|---|
| `GetMaxStackSize()` | скільки копій вміщує слот; `1` без фрагмента `Stackable` |
| `GetUnitWeight()` | вага однієї копії; `0` без фрагмента `Weight` |
| `IsUsable()` | чи має сенс кнопка «Використати» |

---

## Рецепти

Готові комбінації для найтиповіших предметів. Скопіюйте найближчу й змініть значення.

### Зілля здоров'я

```
DA_HealthPotion
├── DisplayName: "Зілля здоров'я"
├── ItemTags: Item.Type.Consumable
└── Fragments:
    ├── Stackable      → MaxStackSize: 10
    └── Consumable     → ConsumeAmount: 1
                         EffectTags: Effect.Heal
```

Саме зцілення реалізуєте ви — плагін лише повідомляє теги ефекту. Як це підключити,
показано в [04 — Фрагменти](04-Fragments.md#consumable--предмет-витрачається).

### Меч

```
DA_IronSword
├── DisplayName: "Залізний меч"
├── ItemTags: Item.Type.Weapon.Melee, Item.Rarity.Common
└── Fragments:
    ├── Equippable     → EquipmentSlot: Equipment.Slot.Weapon.Main
    │                    EquippedActorClass: BP_SwordActor
    │                    AttachSocketName: hand_rSocket
    │                    GrantedStats: { Item.Stat.Damage: 12 }
    └── Durability     → MaxDurability: 100
                         DurabilityPerUse: 1
```

Не стакається (немає `Stackable`), має індивідуальну міцність, з'являється в руці
персонажа.

### Матеріал для крафту

```
DA_CopperOre
├── DisplayName: "Мідна руда"
├── ItemTags: Item.Type.Material
└── Fragments:
    ├── Stackable      → MaxStackSize: 99
    └── Weight         → UnitWeight: 0.5
```

### Боєприпаси для спеціального слота

```
DA_Arrow
├── DisplayName: "Стріла"
├── ItemTags: Item.Type.Ammo
└── Fragments:
    └── Stackable      → MaxStackSize: 50
```

Щоб стріли лягали лише в перші слоти рюкзака, налаштуйте `SlotRestrictions` на
компоненті — див. [05 — Операції з інвентарем](05-Inventory-Operations.md#обмеження-слотів).

### Квестовий предмет

```
DA_SealedLetter
├── DisplayName: "Запечатаний лист"
├── ItemTags: Item.Type.Quest, Item.Property.QuestItem
└── Fragments: (порожньо)
```

Без фрагментів: займає один слот, не стакається, нічого не робить при використанні.
Тег `Item.Property.QuestItem` дозволяє крамниці заблокувати його продаж через
`BlockedItemTags`.

### Смолоскип із пальним

```
DA_Torch
├── DisplayName: "Смолоскип"
├── ItemTags: Item.Type.Tool
└── Fragments:
    ├── Equippable     → EquipmentSlot: Equipment.Slot.Weapon.Offhand
    │                    EquippedActorClass: BP_TorchActor
    └── Consumable     → MaxCharges: 100
                         bDestroyWhenChargesSpent: true
```

Режим зарядів: предмет лишається в слоті, витрачаючи заряди, і зникає, коли пальне
скінчилося.

---

## Перевірка предметів

Плагін перевіряє ассети автоматично — у редакторі під час збереження та на старті
світу в dev-білдах. Він повідомляє про:

- порожню `DisplayName`;
- порожній запис у `Fragments`;
- **два фрагменти одного класу** (другий мовчки перекриває перший);
- `Equippable` без заданого слота;
- `Stackable` із `MaxStackSize < 2`;
- `Durability` із `MaxDurability <= 0` (кожна копія стартувала б зламаною).

Вимикається: **Project Settings → Game → Inventory System → Validate Items On Startup**.

Перевірити все вручну можна командою `DemoInvItems` — вона покаже всі типи предметів у
проєкті.

---

## Куди далі

- Розібратися, що вміє кожен фрагмент: [04 — Фрагменти](04-Fragments.md)
- Навчитися видавати предмети гравцеві:
  [05 — Операції з інвентарем](05-Inventory-Operations.md)
- Наповнити скриню випадковим лутом: [07 — Лут і предмети у світі](07-Loot-And-Pickups.md)
