# 10 — Інтеграція на C++

*[🇬🇧 English](../en/10-CPP-Integration.md) | 🇺🇦 Українська*

Усе, що було до цього розділу, працює й з одного лише Blueprint. Цей розділ — для програмістів, які розширюють плагін на C++.

## Залежності модулів

Плагін складається з чотирьох модулів, усі під `Plugins/QuestSystemDemo/Source/`; додайте до `PublicDependencyModuleNames`/`PrivateDependencyModuleNames` свого `.Build.cs` ті, що реально використовуєте:

| Модуль | Що всередині |
|---|---|
| `QuestSystemDemo` | Core: `UDemoQuestData`/`UDemoQuestObjectiveData`, `UDemoQuestManagerComponent`, `UDemoQuestWorldSubsystem`, `UDemoQuestBlueprintLibrary`, налаштування, валідація. Підключіть `DemoQuestSystemTypes.h`, щоб отримати все одразу. |
| `QuestSystemDemoWorld` | Компоненти та актори для розміщення у світі: `UDemoQuestGiverComponent`/`UDemoQuestReceiverComponent` (від нього успадковується перший приклад у цьому розділі), маркери квестів, `UDemoQuestInteractorComponent`, `UDemoQuestTargetComponent`, готові актори (див. [05](05-World-Components.md)). Залежить від `QuestSystemDemo`; також залежить від `UMG` (через віджет-маркер), тож підключайте його лише там, де ці класи справді потрібні. Підключіть `DemoQuestSystemWorldTypes.h`, щоб отримати все одразу. |
| `QuestSystemDemoDebug` | Опційні dev-інструменти: оверлей налагодження та cheat-команди. Залежить від `QuestSystemDemo` + `UMG`/`Slate`. Підключайте його заголовки напряму (спільного master-include немає). |
| `QuestSystemTests` | Editor-only автотести — не те, від чого залежить проєкт-споживач. |

Власний `HostQuestSystem.Build.cs` демо-хоста — робочий приклад: `QuestSystemDemo` + `QuestSystemDemoWorld` + `QuestSystemDemoDebug` у `PrivateDependencyModuleNames`.

## Кастомна логіка нагород

Повністю розібрана в [02 — Інтеграція без налаштування](02-Zero-Config-Integration.md): успадкуйтеся від `UDemoQuestManagerComponent`, перевизначте `GiveQuestRewards_Implementation` і вкажіть `QuestManagerClass` на свій сабклас у Project Settings. Це єдине перевизначення викликається рівно один раз на квест (захищено від дублів), на тому менеджері, який завершив квест.

```cpp
void UMyQuestManagerComponent::GiveQuestRewards_Implementation(UDemoQuestData* QuestData)
{
    const FDemoQuestReward& Rewards = QuestData->Rewards;

    if (IsPersonalQuestManager())
    {
        if (AMyPlayerState* MyPS = Cast<AMyPlayerState>(GetOwningPlayerState()))
        {
            // RewardAmounts - звичайний TMap<FName, int32>: маршрутизуйте потрібні
            // ключі зі своїх даних квесту туди, де вони щось означають у вашій грі.
            if (const int32* XP = Rewards.RewardAmounts.Find(TEXT("ExperiencePoints")))
            {
                MyPS->AddExperience(*XP);
            }
            if (const int32* Gold = Rewards.RewardAmounts.Find(TEXT("Gold")))
            {
                MyPS->AddGold(*Gold);
            }
            for (const TSubclassOf<AActor>& ItemClass : Rewards.ItemRewards)
            {
                MyPS->GiveItem(ItemClass);
            }
        }
    }
    else if (IsPartyQuestManager())
    {
        // Shared-квест завершився на GameState - нагороджуємо кожного учасника.
        if (const FDemoPartyQuestState* State = FindPartyQuestState(QuestData))
        {
            for (APlayerState* Participant : State->Participants)
            {
                // ... нагородити кожного учасника
            }
        }
    }
}
```

Не розміщуйте логіку видачі нагород в обробнику `OnQuestCompleted` — цей делегат спрацьовує *після* нагород і не захищений від дублів; він існує лише для UI/FX/аналітики.

## Розширення компонентів Giver / Receiver

І `UDemoQuestGiverComponent`, і `UDemoQuestReceiverComponent` виставляють точки кастомізації як `BlueprintNativeEvent`, тож сабклас на C++ перевизначає версію `_Implementation` точно так само, як і будь-яку іншу нативну подію:

```cpp
UCLASS()
class UMyQuestGiverComponent : public UDemoQuestGiverComponent
{
    GENERATED_BODY()
protected:
    virtual bool ShouldOfferQuestToPlayer_Implementation(UDemoQuestData* Quest, APlayerController* Player) const override
    {
        // наприклад, обмеження за рівнем гравця, що зберігається в кастомному PlayerState.
        // Вбудованого поля "рекомендований рівень" немає - використовуємо домовленість через CustomData.
        if (const AMyPlayerState* MyPS = Player ? Player->GetPlayerState<AMyPlayerState>() : nullptr)
        {
            const FString* RequiredLevelStr = Quest->CustomData.Find(TEXT("RequiredLevel"));
            const int32 RequiredLevel = RequiredLevelStr ? FCString::Atoi(**RequiredLevelStr) : 1;
            return MyPS->GetLevel() >= RequiredLevel;
        }
        return Super::ShouldOfferQuestToPlayer_Implementation(Quest, Player);
    }
};
```

## Надсилання подій зі своїх систем

Якщо у вашій грі вже є системи бою, інвентарю чи діалогів, викликайте функції сповіщення напряму з них, а не використовуйте готові актори світу з [05](05-World-Components.md):

```cpp
// В обробці шкоди/смерті, щойно ворог справді помер:
void AMyEnemy::Die(APlayerController* Killer)
{
    if (Killer && Killer->PlayerState)
    {
        UDemoQuestBlueprintLibrary::NotifyKillEvent(Killer->PlayerState, EnemyTypeTag);
    }
}

// У системі інвентарю, щойно предмет справді доданий:
void UMyInventoryComponent::AddItem(FName ItemID, int32 Count)
{
    // ... ваша логіка інвентарю ...
    if (APlayerState* PS = GetOwningPlayerState())
    {
        UDemoQuestBlueprintLibrary::NotifyCollectEvent(PS, ItemID, Count);
    }
}
```

Надсилайте подію лише після того, як базовий ігровий стан справді змінився (ворог дійсно мертвий, предмет дійсно в інвентарі) — квест-система довіряє події як факту й ніяк не може перевірити її на відповідність вашому ігровому стану.

## Читання прогресу з C++ (свій UI)

```cpp
UDemoQuestManagerComponent* Manager = UDemoQuestBlueprintLibrary::GetQuestManager(PlayerState);
for (const FDemoActiveQuest& Quest : Manager->GetActiveQuests())
{
    if (Quest.State != EDemoQuestState::Active) { continue; }

    for (UDemoQuestObjectiveData* Objective : Quest.QuestData->Objectives)
    {
        const FDemoQuestObjectiveProgress Progress = Manager->GetObjectiveProgress(Quest.QuestData, Objective);
        // Progress.CurrentProgress / Progress.TargetProgress / Progress.State
    }
}
```

Оверлей модуля `QuestSystemDemoDebug` (`DemoQuestDebugQuestCard.cpp`/`DemoQuestDebugObjectiveCard.cpp` у `Plugins/QuestSystemDemo/Source/QuestSystemDemoDebug/Private/`) — повноцінний робочий приклад саме цього патерну в масштабі: читання цілей, прогресу й стану кожного квесту для побудови віджетів UMG цілком на C++, без жодного WBP-ассету — варто прочитати, якщо ви будуєте свій quest UI на C++.

## Делегати, а не опитування

Коли треба реагувати на конкретну подію, надавайте перевагу прив'язці до делегатів менеджера (перелічені в [04 — Подієвий прогрес](04-Event-Driven-Progress.md)) замість опитування `GetActiveQuests()` щокадру:

```cpp
QuestManager->OnObjectiveUpdated.AddDynamic(this, &AMyHUD::HandleObjectiveUpdated);
QuestManager->OnQuestCompleted.AddDynamic(this, &AMyHUD::HandleQuestCompleted);
```

## Дивіться також

- [02 — Інтеграція без налаштування](02-Zero-Config-Integration.md) про те, як ваші кастомні класи підключаються без зміни `PlayerState`/`GameState`.
- [06 — Мультиплеєр](06-Multiplayer.md) про API party-квестів, який використовується під час нагородження Shared/Individual квестів.

<!-- doc-footer:start -->
---
*Generated 2026-08-05 11:44 UTC from `Docs/Full/` - do not edit this page directly.*
<!-- doc-footer:end -->
