# 12 — Інтеграція в C++

*[🇬🇧 English](../en/12-Cpp-Integration.md) | 🇺🇦 Українська*

Розділ для проєктів на C++: залежності модулів, персонаж із Enhanced Input, ціль зі звичайною дією, власна ціль захоплення та серверні функції маніпуляції. Приклади використовують модуль гри `MyGame`; замініть назву на свою. Перелік усіх членів класів — у [довіднику API](API-Reference.md).

## Залежності модуля

```csharp
// MyGame.Build.cs
PublicDependencyModuleNames.AddRange(new[]
{
    "Core", "CoreUObject", "Engine", "EnhancedInput",
    "GameplayTags",
    "InteractionSystem",       // UInteractorComponent, IInteractableInterface, claim ledger
    "InteractionSystemWorld"   // manipulation, physics actors, bow, dock, strike
});
```

`InteractionSystem` — мінімальна залежність: інтерфейс, interactor і реєстр. `InteractionSystemWorld` потрібен, якщо ви використовуєте готові актори чи компоненти маніпуляції. `GameplayTags` потрібен, якщо ваші публічні заголовки використовують `FGameplayTagContainer`.

## Персонаж для desktop

```cpp
// MyCharacter.h
#pragma once

#include "CoreMinimal.h"
#include "GameFramework/Character.h"
#include "InteractionSystemTypes.h"
#include "MyCharacter.generated.h"

class UCameraComponent;
class UInputAction;
class UInputMappingContext;
class UInteractorComponent;
struct FInputActionValue;

UCLASS()
class MYGAME_API AMyCharacter : public ACharacter
{
    GENERATED_BODY()

public:
    AMyCharacter();

    virtual void NotifyControllerChanged() override;
    virtual void SetupPlayerInputComponent(UInputComponent* InInputComponent) override;

protected:
    virtual void BeginPlay() override;

    UPROPERTY(VisibleAnywhere, Category = "Interaction")
    TObjectPtr<UCameraComponent> Camera;

    UPROPERTY(VisibleAnywhere, Category = "Interaction")
    TObjectPtr<UInteractorComponent> Interactor;

    UPROPERTY(EditDefaultsOnly, Category = "Input")
    TObjectPtr<UInputMappingContext> InteractionContext;

    UPROPERTY(EditDefaultsOnly, Category = "Input")
    TObjectPtr<UInputAction> UseAction;

    UPROPERTY(EditDefaultsOnly, Category = "Input")
    TObjectPtr<UInputAction> ThrowAction;

    UPROPERTY(EditDefaultsOnly, Category = "Interaction")
    float ThrowSpeed = 900.f;

private:
    void HandleUse(const FInputActionValue& InValue);
    void HandleThrow(const FInputActionValue& InValue);

    UFUNCTION()
    void HandleActionResult(AActor* InTarget, EInteractionAction InAction, bool bInAccepted,
                            EInteractionRejectReason InReason);
};
```

```cpp
// MyCharacter.cpp
#include "MyCharacter.h"
#include "Camera/CameraComponent.h"
#include "Components/CapsuleComponent.h"
#include "EnhancedInputComponent.h"
#include "EnhancedInputSubsystems.h"
#include "Engine/LocalPlayer.h"
#include "GameFramework/PlayerController.h"
#include "InteractionPhysicsActor.h"
#include "InteractorComponent.h"

AMyCharacter::AMyCharacter()
{
    bReplicates = true;

    Camera = CreateDefaultSubobject<UCameraComponent>(TEXT("Camera"));
    Camera->SetupAttachment(GetCapsuleComponent());
    Camera->SetRelativeLocation(FVector(0, 0, 64));
    Camera->bUsePawnControlRotation = true;

    // One interactor follows the controller view; a held item sits in front of the camera.
    Interactor = CreateDefaultSubobject<UInteractorComponent>(TEXT("Interactor"));
    Interactor->SetupAttachment(Camera);
    Interactor->GrabOffset = FVector(70, 0, -15);
}

void AMyCharacter::BeginPlay()
{
    Super::BeginPlay();
    Interactor->OnActionResult.AddDynamic(this, &AMyCharacter::HandleActionResult);
}

void AMyCharacter::NotifyControllerChanged()
{
    Super::NotifyControllerChanged();
    // Possession can reach a client after BeginPlay, so the local mapping is added here.
    if (const APlayerController* PlayerController = Cast<APlayerController>(GetController()))
    {
        if (auto* InputSubsystem = ULocalPlayer::GetSubsystem<UEnhancedInputLocalPlayerSubsystem>(PlayerController->GetLocalPlayer()))
        {
            InputSubsystem->AddMappingContext(InteractionContext, 0);
        }
    }
}

void AMyCharacter::SetupPlayerInputComponent(UInputComponent* InInputComponent)
{
    Super::SetupPlayerInputComponent(InInputComponent);
    if (auto* EnhancedInput = Cast<UEnhancedInputComponent>(InInputComponent))
    {
        EnhancedInput->BindAction(UseAction, ETriggerEvent::Started, this, &AMyCharacter::HandleUse);
        EnhancedInput->BindAction(ThrowAction, ETriggerEvent::Started, this, &AMyCharacter::HandleThrow);
    }
}

void AMyCharacter::HandleUse(const FInputActionValue& InValue)
{
    if (Interactor->GetHeldActor())
    {
        // The server routes this to the held item's Use and the manipulation OnUsed event.
        Interactor->RequestInteraction();
        return;
    }
    Interactor->RefreshFocus();
    const auto* PhysicsItem = Cast<AInteractionPhysicsActor>(Interactor->GetFocusedActor());
    if (PhysicsItem && PhysicsItem->bCanBeGrabbed)
    {
        Interactor->RequestGrab();
    }
    else
    {
        Interactor->RequestInteraction();
    }
}

void AMyCharacter::HandleThrow(const FInputActionValue& InValue)
{
    if (Interactor->GetHeldActor())
    {
        // Linear velocity in cm/s and angular velocity in degrees/s; the server clamps both.
        Interactor->RequestRelease(Camera->GetForwardVector() * ThrowSpeed, FVector(0, 120, 0));
    }
}

void AMyCharacter::HandleActionResult(AActor* InTarget, EInteractionAction InAction, bool bInAccepted,
                                      EInteractionRejectReason InReason)
{
    if (!bInAccepted && InReason == EInteractionRejectReason::Claimed)
    {
        // Another player holds the target: tell the player instead of failing silently.
    }
}
```

`UseAction`, `ThrowAction` і `InteractionContext` призначте в Blueprint-нащадку персонажа. Імена параметрів мають префікси `In`: `InputComponent` — член `AActor`, і параметр з таким самим ім'ям ховав би його, що на Windows (MSVC) є помилкою компіляції. З тієї ж причини не називайте локальні змінні `Owner`, `Tags` чи `Manipulation` усередині акторів, які вже мають такі члени.

## Ціль зі звичайною дією

```cpp
// MyDoor.h
#pragma once

#include "CoreMinimal.h"
#include "GameFramework/Actor.h"
#include "InteractableInterface.h"
#include "MyDoor.generated.h"

UCLASS()
class MYGAME_API AMyDoor : public AActor, public IInteractableInterface
{
    GENERATED_BODY()

public:
    AMyDoor();

    virtual void GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const override;
    virtual bool CanInteract_Implementation(UInteractorComponent* InInteractor, EInteractionAction InAction) const override;
    virtual void Interaction_Implementation(UInteractorComponent* InInteractor) override;

private:
    UPROPERTY(VisibleAnywhere, Category = "Door")
    TObjectPtr<UStaticMeshComponent> Leaf;

    UPROPERTY(ReplicatedUsing = OnRep_Open)
    bool bOpen = false;

    UFUNCTION()
    void OnRep_Open();
};
```

```cpp
// MyDoor.cpp
#include "MyDoor.h"
#include "Components/StaticMeshComponent.h"
#include "Net/UnrealNetwork.h"

AMyDoor::AMyDoor()
{
    bReplicates = true;
    Leaf = CreateDefaultSubobject<UStaticMeshComponent>(TEXT("Leaf"));
    Leaf->SetMobility(EComponentMobility::Movable);
    SetRootComponent(Leaf);
}

void AMyDoor::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);
    DOREPLIFETIME(AMyDoor, bOpen);
}

bool AMyDoor::CanInteract_Implementation(UInteractorComponent* InInteractor, EInteractionAction InAction) const
{
    // Clients call this for focus too, so it must not change state.
    return InAction == EInteractionAction::Interact;
}

void AMyDoor::Interaction_Implementation(UInteractorComponent* InInteractor)
{
    // Runs on the server after the trace, tags and CanInteract were rechecked there.
    bOpen = !bOpen;
    OnRep_Open();
}

void AMyDoor::OnRep_Open()
{
    Leaf->SetRelativeRotation(FRotator(0, bOpen ? 90.f : 0.f, 0));
}
```

Gameplay Tags для фільтрів повертайте з `GetInteractionTags_Implementation`, а локальну підсвітку — з `OnFocusStart_Implementation` і `OnFocusEnd_Implementation`.

Якщо ціль з інтерфейсом також можна брати (має компонент цілі захоплення), сервер повідомить її про підбір і відпускання:

```cpp
// In the class declaration
virtual void OnGrab_Implementation(UInteractorComponent* InInteractor) override;
virtual void OnRelease_Implementation(UInteractorComponent* InInteractor, EInteractionReleaseReason InReason,
                                      FVector InLinearVelocity, FVector InAngularVelocity) override;
```

`OnGrab` надходить, коли першу руку прийнято, `OnRelease` — коли відпустила остання; швидкості вже обмежені сервером і нульові для будь-якої причини, крім `Released`.

## Мережеві межі в коді

Типові межі живуть у `UInteractionSystemSettings` (**Project Settings → Plugins → Interaction System**). Окремий interactor перевизначає значення парою «прапорець + значення»:

```cpp
// Server and clients must agree; the server's value is final.
Interactor->bOverrideMaxThrowSpeed = true;
Interactor->MaxThrowSpeed = 3200.f;

const float Reach = Interactor->GetMaxOriginDistance(); // effective value: override or project default
const float DefaultTimeout = GetDefault<UInteractionSystemSettings>()->PoseTimeout;
```

Для `GetDefault<UInteractionSystemSettings>()` підключіть `InteractionSystemSettings.h`; модуль `InteractionSystem` уже тягне потрібний `DeveloperSettings`.

## Власна ціль захоплення

Готова маніпуляція покриває більшість предметів. Якщо рух потрібно реалізувати інакше (важіль на шарнірі, мотузка, власна фізика), є два шляхи: Blueprint-нащадок `UInteractionBlueprintGrabTargetComponent` без C++ ([11](11-Blueprint-Integration.md#власна-ціль-захоплення-в-blueprint)) або C++-нащадок `UInteractionGrabTargetComponent`. Контракт:

- `CanBeginGrip` викликається локально для фокуса й на сервері перед хватом — читайте лише replicated-стан;
- `BeginGrip` викликається на сервері після резервування; `false` скасовує резерв;
- `EndGrip` прибирає лише цю руку; вимушене завершення (причина не `Released`) не повинно кидати предмет;
- не створюйте власного реєстру власників: ним є `UInteractionClaimSubsystem`.

```cpp
// MyLeverGrabTarget.h
#pragma once

#include "CoreMinimal.h"
#include "InteractionGrabTargetComponent.h"
#include "MyLeverGrabTarget.generated.h"

UCLASS(ClassGroup = (Interaction), meta = (BlueprintSpawnableComponent))
class MYGAME_API UMyLeverGrabTarget : public UInteractionGrabTargetComponent
{
    GENERATED_BODY()

public:
    UMyLeverGrabTarget();

    virtual void GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const override;
    virtual bool CanBeginGrip(const UInteractorComponent* InHand) const override;
    virtual bool BeginGrip(UInteractorComponent* InHand) override;
    virtual void EndGrip(UInteractorComponent* InHand, EInteractionReleaseReason InReason,
                         const FVector& InLinearVelocity, const FVector& InAngularVelocity) override;

private:
    /** Replicated so that client focus agrees with the server. */
    UPROPERTY(Replicated)
    TObjectPtr<UInteractorComponent> HoldingHand;
};
```

```cpp
// MyLeverGrabTarget.cpp
#include "MyLeverGrabTarget.h"
#include "GameFramework/Actor.h"
#include "InteractorComponent.h"
#include "Net/UnrealNetwork.h"

UMyLeverGrabTarget::UMyLeverGrabTarget()
{
    SetIsReplicatedByDefault(true);
}

void UMyLeverGrabTarget::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
    Super::GetLifetimeReplicatedProps(OutLifetimeProps);
    DOREPLIFETIME(UMyLeverGrabTarget, HoldingHand);
}

bool UMyLeverGrabTarget::CanBeginGrip(const UInteractorComponent* InHand) const
{
    return IsActive() && IsValid(InHand) && InHand->HasValidTracking() && !HoldingHand;
}

bool UMyLeverGrabTarget::BeginGrip(UInteractorComponent* InHand)
{
    if (!GetOwner()->HasAuthority() || !CanBeginGrip(InHand))
    {
        return false;
    }
    HoldingHand = InHand;
    return true;
}

void UMyLeverGrabTarget::EndGrip(UInteractorComponent* InHand, EInteractionReleaseReason InReason,
                                 const FVector& InLinearVelocity, const FVector& InAngularVelocity)
{
    if (HoldingHand == InHand)
    {
        HoldingHand = nullptr;
    }
}
```

Рух важеля реалізуйте самі, наприклад у `TickComponent` за `HoldingHand->GetGripTransform()` на сервері, з реплікацією результату. Щоб примусово звільнити руку, на сервері викличте `HoldingHand->CancelTarget(GetOwner(), EInteractionReleaseReason::TargetInvalid)`: interactor викличе ваш `EndGrip` і зніме резерв. Для `Use` утримуваного предмета перевизначте `Use`.

## Серверні функції маніпуляції

Функції `UInteractionManipulationComponent`, що змінюють стан, виконуються лише на сервері; у Blueprint вони позначені як authority-only. Приклад розміщення предмета на полиці:

```cpp
#include "InteractionManipulationComponent.h"

void AMyShelf::PlaceItem(AActor* InItem, USceneComponent* InSlot)
{
    auto* ItemManipulation = InItem->FindComponentByClass<UInteractionManipulationComponent>();
    if (!HasAuthority() || !ItemManipulation || ItemManipulation->GetHandCount() > 0)
    {
        return;
    }
    // Stop free simulation, snap the item to the slot with its own scale, then let the snapshot own the attachment.
    ItemManipulation->SetFreeSimulation(false);
    FTransform SeatedPose = InSlot->GetComponentTransform();
    SeatedPose.SetScale3D(InItem->GetActorScale3D());
    ItemManipulation->GetManipulatedComponent()->SetWorldTransform(SeatedPose, false, nullptr,
                                                                   ETeleportType::TeleportPhysics);
    ItemManipulation->AttachToSurface(InSlot);
}
```

| Функція | Призначення |
|---|---|
| `AttachToSurface(Parent, Socket)` | Прикріпити вільний предмет до компонента іншого актора, зберігши світову позу. Повторне захоплення знімає прикріплення |
| `SetFreeSimulation(bSimulate, Velocity, AngularVelocity)` | Зняти прикріплення й зовнішній драйвер і задати симуляцію, наприклад щоб запустити предмет пасткою |
| `SetExternalDriver(Driver)` / `SetExternalPose(Pose)` | Передати рух іншому актору: колесо на дверях, стріла на тятиві |
| `SetGrabEnabled(bEnabled)` | Заборонити чи дозволити захоплення; вимкнення відпускає поточні руки, значення реплікується |
| `ResetManipulation()` / `CancelAllGrips()` | Повернути предмет додому або звільнити всі руки без кидка |
| `SetHomeTransform(Transform)` / `GetHomeTransform()` | Змінити або прочитати домашню позу для `ResetManipulation` |
| `SyncTransform()` | Негайно надіслати клієнтам позу, яку змінив ваш код |
| `ValidateSetup(OutReason)` | Перевірити налаштування; корисно в `BeginPlay` з виведенням причини в лог |

## Підписка на події маніпуляції

Приклад для `AMyRifle`, нащадка `AInteractionPhysicsActor` (член `Manipulation` уже є в базовому класі):

```cpp
void AMyRifle::BeginPlay()
{
    Super::BeginPlay();
    if (HasAuthority())
    {
        Manipulation->OnUsed.AddDynamic(this, &AMyRifle::HandleRifleTrigger);
        Manipulation->OnGripEnded.AddDynamic(this, &AMyRifle::HandleRifleReleased);
    }
}

void AMyRifle::HandleRifleTrigger(UInteractorComponent* InHand)
{
    // Server-side shot; replicate effects with a multicast or a replicated counter.
}

void AMyRifle::HandleRifleReleased(UInteractorComponent* InHand, bool bInLast, EInteractionReleaseReason InReason,
                                   FVector InLinearVelocity, FVector InAngularVelocity)
{
    if (bInLast && InReason == EInteractionReleaseReason::Released)
    {
        // Intentional final release; tracking loss and cancellation never reach this branch.
    }
}
```

Обробники динамічних делегатів мають бути `UFUNCTION()`. Не повторюйте імен `UFUNCTION` базового класу: `AInteractionPhysicsActor` уже має `HandlePhysicsHit`, `HandlePhysicsSleep` і `MulticastImpactSound`, і UHT не дозволить оголосити їх у нащадку ще раз.

## Читання власника на сервері

```cpp
#include "InteractionClaimSubsystem.h"

if (const auto* Claims = GetWorld()->GetSubsystem<UInteractionClaimSubsystem>())
{
    const UInteractorComponent* ClaimingHand = Claims->FindClaimant(Item); // first hand or nullptr
    const APawn* OwningPawn = Claims->FindOwningPawn(Item);                // owner of a grip or a linked reservation
    const TArray<AActor*> HeldTargets = Claims->GetClaimedTargets();       // everything held or reserved now
}
```

Реєстр не реплікується: на клієнті використовуйте `GetHeldActor()` interactor або `GetPrimaryHand()` маніпуляції.

## Математика для власних тестів

`FInteractionRules` і `FInteractionGripMath` не звертаються до світу, тому підходять для швидких unit-тестів власних профілів:

```cpp
const FTransform First(FVector(0, -10, 0));
const FTransform Second(FVector(0, 10, 0));
const FTransform Object(FQuat::Identity, FVector(40, 0, 0), FVector::OneVector);
const FInteractionGripBaseline Baseline = FInteractionGripMath::Capture(Object, First, &Second);

const FTransform WideFirst(FVector(0, -20, 0));
const FTransform WideSecond(FVector(0, 20, 0));
// Doubling the hand separation doubles the uniform scale around the midpoint.
const FTransform Scaled = FInteractionGripMath::Solve(Baseline, WideFirst, &WideSecond, true, true, 0.25f, 4.f, Object);
```
