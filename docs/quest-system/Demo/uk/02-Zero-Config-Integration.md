# 02 — Інтеграція без налаштування

*[🇬🇧 English](../en/02-Zero-Config-Integration.md) | 🇺🇦 Українська*

Щоб користуватися QuestSystemDemo, не потрібен кастомний клас `PlayerState`, `GameState` чи `GameMode`. Цей розділ пояснює, що робить усю проводку за вас і як налаштувати її під себе, коли це знадобиться.

## UDemoQuestWorldSubsystem

`UDemoQuestWorldSubsystem` — це `UWorldSubsystem`, він існує автоматично в кожному ігровому світі, жодного налаштування не потрібно. На сервері (одиночна standalone-сесія теж вважається сервером) він:

- створює `UDemoQuestManagerComponent` на **GameState** (party-менеджер);
- створює `UDemoQuestManagerComponent` на **кожному `PlayerState`** у міру його спавну — це покриває вхід у гру, пізнє підключення, seamless travel і PIE;
- зараховує нових підключених гравців до активних party-квестів із увімкненим `bAutoEnrollNewParticipants` (див. [06 — Мультиплеєр](06-Multiplayer.md));
- опційно валідує весь граф залежностей квестів під час старту й пише результат у лог (лише в dev-збірках).

Якщо у `PlayerState` чи `GameState` **вже є** `UDemoQuestManagerComponent` — бо ви розмістили його вручну в кастомному класі чи Blueprint, — сабсистема його не чіпає. Ручне налаштування й налаштування без коду можуть співіснувати.

```cpp
// Отримати сабсистему напряму, якщо потрібно:
UDemoQuestWorldSubsystem* Subsystem = UDemoQuestWorldSubsystem::Get(this);
UDemoQuestManagerComponent* PartyManager = Subsystem->GetPartyQuestManager();
UDemoQuestManagerComponent* MyManager = UDemoQuestWorldSubsystem::GetPlayerQuestManager(MyPlayerState);
```

У більшості випадків замість цього ви користуватиметеся `UDemoQuestBlueprintLibrary` (див. [08](08-Blueprint-Library-Reference.md)) — вона огортає ті самі пошуки зрозумілішими іменами (`GetQuestManager`, `GetPartyQuestManager`).

## UDemoQuestSystemSettings

Project Settings → **Game → Quest System (Demo)** (зберігається в `DefaultGame.ini` під `[/Script/QuestSystemDemo.DemoQuestSystemSettings]`):

| Властивість | За замовчуванням | Значення |
|----------|---------|---------|
| `bAutoCreatePlayerQuestManagers` | `true` | Створювати менеджер на кожному `PlayerState`. |
| `bAutoCreatePartyQuestManager` | `true` | Створювати менеджер на `GameState`. Обов'язковий для Shared/Individual квестів. |
| `QuestManagerClass` | `UDemoQuestManagerComponent` | Клас, який створює сабсистема. Вкажіть тут свій сабклас — див. нижче. |
| `bValidateQuestsOnStartup` | `true` | Запускати валідацію залежностей під час старту світу (лише не-shipping збірки). |

Пороги попередження таймера задаються для кожного ассету квесту/цілі окремо (`TimerWarningThreshold`, `0` = без попередження), а не одним спільним проєктним значенням — див. [03 — Створення квестів](03-Authoring-Quests.md#таймерні-квести).

```ini
; Приклад: DefaultGame.ini
[/Script/QuestSystemDemo.DemoQuestSystemSettings]
bAutoCreatePartyQuestManager=false
```

## Перевизначення логіки нагород

`GiveQuestRewards()` — це `BlueprintNativeEvent` на `UDemoQuestManagerComponent`, природне місце для видачі досвіду, золота чи предметів. Оскільки менеджер створює сабсистема, для доступу до нього не потрібно успадковуватися від `PlayerState` — натомість успадковуйтеся від самого **компонента** й вкажіть `QuestManagerClass` на свій сабклас. Підійде як сабклас на C++, так і на Blueprint — обирайте, що зручніше вашій команді.

### Варіант на C++

```cpp
// MyQuestManagerComponent.h
UCLASS()
class UMyQuestManagerComponent : public UDemoQuestManagerComponent
{
    GENERATED_BODY()
protected:
    virtual void GiveQuestRewards_Implementation(UDemoQuestData* QuestData) override;
};

// MyQuestManagerComponent.cpp
void UMyQuestManagerComponent::GiveQuestRewards_Implementation(UDemoQuestData* QuestData)
{
    if (APlayerState* PS = GetOwningPlayerState())
    {
        if (AMyPlayerState* MyPS = Cast<AMyPlayerState>(PS))
        {
            // RewardAmounts - це звичайний TMap<FName, int32>: плагін не має
            // вбудованого поняття "золото" чи "досвід", тож маршрутизуйте
            // потрібні ключі зі своїх даних квесту туди, де вони щось означають у вашій грі.
            if (const int32* XP = QuestData->Rewards.RewardAmounts.Find(TEXT("ExperiencePoints")))
            {
                MyPS->AddExperience(*XP);
            }
            if (const int32* Gold = QuestData->Rewards.RewardAmounts.Find(TEXT("Gold")))
            {
                MyPS->AddGold(*Gold);
            }
        }
    }
    // Викликається і на менеджері PlayerState (Personal/Individual), і на
    // менеджері GameState (Shared) - GetOwningPlayerState() на party-менеджері
    // поверне nullptr, тож для нагородження всіх учасників Shared-квесту
    // розгалужуйтеся за IsPartyQuestManager().
}
```

Потім укажіть **QuestManagerClass = MyQuestManagerComponent** у Project Settings. Більше нічого міняти не потрібно — це працює незалежно від того, чи опиниться компонент на ванільному `PlayerState`, чи на кастомному.

### Варіант на Blueprint (без C++)

Те, що `GiveQuestRewards` — це `BlueprintNativeEvent`, означає, що Blueprint-клас може перевизначити його точно так само, як і сабклас на C++ — жодної особливої обробки з боку рушія не потрібно:

1. Content Browser → **Add → Blueprint Class**. У діалозі вибору батька увімкніть **All Classes** і знайдіть **Quest Manager Component** — оберіть його як батька.
2. Назвіть, наприклад, `BP_MyQuestManagerComponent`, і відкрийте.
3. В Event Graph правою кнопкою → **Add Event** → знайдіть **Give Quest Rewards**. З'явиться нода `Event Give Quest Rewards` з вихідним піном `Quest Data`.
4. Проведіть логіку нагород звідти: `Break` структури `Rewards` піна `Quest Data`, щоб отримати `Item Rewards` / `Reward Amounts`, зробіть `Find` за ключем у `Reward Amounts` (наприклад `"ExperiencePoints"`, `"Gold"` або як ваш проєкт називає свої ресурси), щоб дістати суму, потім викличте потрібні функції вашої системи характеристик чи інвентарю (наприклад, `Get Player State` → каст до свого `PlayerState` → `Add Experience`).
5. Project Settings → **Game → Quest System (Demo)** → вкажіть `Quest Manager Class` = `BP_MyQuestManagerComponent`.

Реалізація за замовчуванням на C++ (лише логування) не виконається, доки ви явно не додасте **Call to Parent Function** на ноді події — пропустити його нормально, ви не втрачаєте нічого, крім рядка в лозі.

## Ручне розміщення (відмова від автоматики)

Якщо хочете повний контроль, вимкніть відповідне налаштування `bAutoCreate*` і створіть компонент самі — точно так само, як до появи сабсистеми:

```cpp
AMyPlayerState::AMyPlayerState()
{
    QuestManagerComponent = CreateDefaultSubobject<UDemoQuestManagerComponent>(TEXT("QuestManagerComponent"));
    QuestManagerComponent->SetIsReplicated(true);
}
```

Обидва підходи дають ідентичний, повністю робочий компонент — інтеграція без налаштування просто рекомендується за замовчуванням.

<!-- doc-footer:start -->
---
*Generated 2026-08-05 11:23 UTC from `Docs/Full/` - do not edit this page directly.*
<!-- doc-footer:end -->
