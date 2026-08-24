# 11 — Рецепти

*[🇬🇧 English](../en/11-Recipes.md) | 🇺🇦 Українська*

Готові розв'язки типових задач. Кожен — самодостатній: скопіюйте найближчий і
підправте під себе.

Усе показане працює і в одиночній грі, і в мультиплеєрі без змін.

---

## Крафт

Витратити матеріали й видати результат. Головне тут — **не списати наполовину**.

```cpp
bool AMyCraftingStation::TryCraft(AActor* Player, const FMyRecipe& Recipe)
{
    UISInventoryComponent* Inv = UISInventoryBlueprintLibrary::GetInventoryFor(Player);
    if (!Inv)
    {
        return false;
    }

    // 1. Перевірити ВСЕ, перш ніж витрачати бодай щось.
    for (const TPair<UISItemDefinition*, int32>& Ingredient : Recipe.Ingredients)
    {
        if (!Inv->HasItem(Ingredient.Key, Ingredient.Value))
        {
            return false;   // бракує - нічого не чіпаємо
        }
    }

    // 2. Перевірити, що результат матиме куди лягти.
    if (Inv->CanAcceptItemCount(Recipe.Result, Recipe.ResultCount) < Recipe.ResultCount)
    {
        return false;   // інакше матеріали зникли б, а меч - ні
    }

    // 3. Тепер списувати безпечно.
    for (const TPair<UISItemDefinition*, int32>& Ingredient : Recipe.Ingredients)
    {
        int32 Remainder = 0;
        Inv->TryRemoveItem(Ingredient.Key, Ingredient.Value, Remainder);
    }

    int32 Remainder = 0;
    Inv->TryAddItem(Recipe.Result, Recipe.ResultCount, Remainder);
    return true;
}
```

Два кроки перевірки — не зайва обережність. Без першого гравець втратить частину
матеріалів на невдалому крафті; без другого — втратить усі, а результат не влізе.

---

## Торгівля

Купівля — це перевірка валюти, обмін, і жодного проміжного стану, у якому гравець уже
заплатив, але ще не отримав.

```cpp
bool AMyVendor::TryBuy(AActor* Buyer, int32 VendorSlot, int32 Count)
{
    UISInventoryComponent* BuyerInv = UISInventoryBlueprintLibrary::GetInventoryFor(Buyer);
    if (!BuyerInv || !Stock)
    {
        return false;
    }

    UISItemInstance* Goods = Stock->GetItemAtSlot(VendorSlot);
    if (!Goods)
    {
        return false;
    }

    const int32 Price = GetPrice(Goods->Definition) * Count;

    // Гроші є? Місце є?
    if (!BuyerInv->HasItem(CurrencyDefinition, Price))
    {
        return false;
    }
    if (BuyerInv->CanAcceptItemCount(Goods->Definition, Count) < Count)
    {
        return false;
    }

    // Спершу списати валюту, потім передати товар.
    int32 Remainder = 0;
    BuyerInv->TryRemoveItem(CurrencyDefinition, Price, Remainder);
    BuyerInv->TryTransferFrom(Stock, VendorSlot, -1, Count);

    return true;
}
```

**Продаж** — дзеркально: `BuyerInv->TryTransferTo(Stock, PlayerSlot)` і видати валюту.

Щоб крамниця не приймала квестові предмети, налаштуйте фільтри на її інвентарі:

```
Stock->AllowedItemTags: Item.Type.Weapon, Item.Type.Armor
Stock->BlockedItemTags: Item.Property.QuestItem
```

Тоді `TryTransferTo` відхилить непридатне сам, без жодної перевірки у вашому коді.

---

## Валюта як предмет

Найпростіший спосіб — звичайний предмет із великим стеком:

```
DA_Gold
├── ItemTags: Item.Type.Currency
└── Fragments: Stackable → MaxStackSize: 99999
```

Тоді гроші живуть в інвентарі, зберігаються разом із ним і показуються тим самим UI.

```cpp
const int32 Gold = Inventory->GetItemCount(GoldDefinition);
```

Якщо валюта не має займати слот — тримайте її поза інвентарем, у своєму компоненті.
Плагін на цьому не наполягає.

---

## Хотбар (панель швидкого доступу)

Найпростіше рішення: хотбар — це **перші N слотів** того самого інвентаря.

```cpp
// Гравець натиснув «3»
Inventory->TryUseItem(2, GetPawn());   // слоти 0..N-1 = кнопки 1..N
```

Нічого синхронізувати не треба: слот один, UI показує його двічі.

Якщо хотбар має приймати лише певне — додайте обмеження слотів:

```
SlotRestrictions:
  └── [0]: FirstSlot 0, LastSlot 4, RequiredTags: Item.Type.Consumable
```

Складніший варіант — хотбар як окремі **посилання** на слоти основного інвентаря
(масив індексів у вашому UI). Тоді предмет лишається на місці, а хотбар лише вказує на
нього. Плагін цьому не заважає — це чисто ваш стан UI.

---

## Лутання трупа

Труп — це актор із компонентом інвентаря. Нічого спеціального.

```cpp
void AMyCharacter::OnDeath()
{
    // Не видаляємо актор одразу - його інвентар і є здобиччю.
    SetLifeSpan(120.0f);
    bIsLootable = true;
}
```

Гравець забирає так само, як зі скрині:

```cpp
PlayerInventory->TryTransferFrom(CorpseInventory, SlotIndex);
```

Щоб замість цього **висипати все на землю**:

```cpp
void AMyCharacter::DropEverything()
{
    if (!HasAuthority()) return;

    for (const FISInventoryEntry& Entry : Inventory->GetAllEntries())
    {
        UISItemInstance* Item = Entry.Instance;
        const int32 Slot = Entry.SlotIndex;

        // Забрати з інвентаря, потім віддати пікапу - предмет зберігає стан.
        Inventory->TryRemoveItemFromSlot(Slot, Item->StackCount);
        AISItemPickup::SpawnForItem(this, Item, GetActorLocation() + FMath::VRand() * 100.0f);
    }
}
```

> Обходьте **копію** списку, якщо змінюєте інвентар усередині циклу. Простіше —
> зібрати індекси наперед, потім діяти.

---

## Викинути предмет

```cpp
void AMyCharacter::DropItem(int32 SlotIndex)
{
    UISItemInstance* Item = Inventory->GetItemAtSlot(SlotIndex);
    if (!Item) return;

    const FVector DropAt = GetActorLocation() + GetActorForwardVector() * 150.0f;

    Inventory->TryRemoveItemFromSlot(SlotIndex, Item->StackCount);
    AISItemPickup::SpawnForItem(this, Item, DropAt);
}
```

Порядок важливий: спершу забрати з інвентаря, потім передати пікапу. Пікап **бере
володіння** самим об'єктом, тому меч на 12/100 міцності лежатиме на землі саме таким.

Заборонити викидати квестове — перевірте тег:

```cpp
if (Item->Definition->ItemTags.HasTag(QuestItemTag))
{
    ShowToast(NSLOCTEXT("UI", "CantDrop", "Це не можна викинути."));
    return;
}
```

---

## Підбирання при дотику

```cpp
void AMyPickupTrigger::NotifyActorBeginOverlap(AActor* Other)
{
    Super::NotifyActorBeginOverlap(Other);

    if (!HasAuthority()) return;   // збирає сервер

    if (UISInventoryBlueprintLibrary::HasInventory(Other))
    {
        TryCollect(Other);
    }
}
```

`HasInventory` рятує від зайвої роботи, коли в об'єм зайшло щось без інвентаря.

---

## Нагорода за квест чи подію

Найкоротший шлях — без пошуку компонента:

```cpp
const int32 Given = UISInventoryBlueprintLibrary::GiveItemTo(Player, RewardDef, 3);

if (Given < 3)
{
    // Влізло не все - висипати решту під ноги.
    SpawnRemainderAsPickup(Player, RewardDef, 3 - Given);
}
```

---

## Стартовий набір

Без коду взагалі: заповніть `StartingItems` на компоненті інвентаря в Blueprint
персонажа. Предмети видадуться на сервері при старті.

Якщо набір залежить від класу персонажа — простіше кодом:

```cpp
void AMyCharacter::BeginPlay()
{
    Super::BeginPlay();

    if (HasAuthority())
    {
        for (const TPair<UISItemDefinition*, int32>& Entry : ClassStartingKit)
        {
            int32 Remainder = 0;
            Inventory->TryAddItem(Entry.Key, Entry.Value, Remainder);
        }
    }
}
```

---

## Ремонт спорядження

```cpp
void AMyBlacksmith::Repair(UISItemInstance* Item)
{
    if (!Item || !Item->Definition) return;

    if (const UISFragment_Durability* Dur = Item->Definition->FindFragment<UISFragment_Durability>())
    {
        Dur->Repair(Item);   // від'ємне значення = повний ремонт
    }
}
```

Ціну зручно рахувати від того, скільки бракує:

```cpp
const float Missing = Dur->MaxDurability - Dur->GetCurrentDurability(Item);
const int32 Cost = FMath::CeilToInt(Missing * CostPerPoint);
```

---

## Розширення компонента

Коли потрібна власна поведінка на кожному інвентарі — успадкуйтеся:

```cpp
UCLASS()
class UMyInventoryComponent : public UISInventoryComponent
{
    GENERATED_BODY()

public:
    UMyInventoryComponent()
    {
        MaxSlots = 40;
        MaxWeight = 120.0f;
    }
};
```

Далі або ставте свій клас у Blueprint персонажа, або вкажіть його в
**Project Settings → Game → Inventory System → Inventory Component Class**, щоб
автостворення користувалося саме ним.

Те саме працює для `UISEquipmentComponent` і для будь-якого фрагмента.

---

## Кілька інвентарів на одному акторі

Просто додайте кілька компонентів — рюкзак, гаманець, сумка для квестового:

```cpp
Backpack   = CreateDefaultSubobject<UISInventoryComponent>(TEXT("Backpack"));
QuestBag   = CreateDefaultSubobject<UISInventoryComponent>(TEXT("QuestBag"));

QuestBag->AllowedItemTags.AddTag(QuestItemTag);
```

Пам'ятайте: `GetInventoryFor` поверне **перший знайдений**. Якщо інвентарів кілька,
або зберігайте посилання явно, або реалізуйте `IISInventoryInterface` і поверніть із
нього той, що вважаєте головним:

```cpp
virtual UISInventoryComponent* GetInventoryComponent() const override { return Backpack; }
```

Інтерфейс має пріоритет над пошуком — саме для цього випадку він і існує.

---

## Куди далі

- Намалювати це все: [08 — Побудова інтерфейсу](08-Building-UI.md)
- Написати власний фрагмент: [04 — Фрагменти](04-Fragments.md#власний-фрагмент)
- Перевірити, що вийшло: [13 — Відлагодження](13-Debugging.md)
