# 12 — C++ integration

*🇬🇧 English | [🇺🇦 Українська](../uk/12-Cpp-Integration.md)*

This chapter is for C++ projects: module dependencies, a character with Enhanced Input, a target with an ordinary action, a custom grab target and the server-side manipulation functions. The examples use the game module `MyGame`; replace the name with yours. The list of all class members is in the [API reference](API-Reference.md).

## Module dependencies

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

`InteractionSystem` is the minimal dependency: the interface, the interactor and the ledger. `InteractionSystemWorld` is needed if you use the ready-made actors or manipulation components. `GameplayTags` is needed if your public headers use `FGameplayTagContainer`.

## A desktop character

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

Assign `UseAction`, `ThrowAction` and `InteractionContext` in the character's Blueprint subclass. Parameter names have the prefix `In`: `InputComponent` is a member of `AActor`, and a parameter with the same name would hide it, which is a compilation error on Windows (MSVC). For the same reason do not name local variables `Owner`, `Tags` or `Manipulation` inside actors that already have such members.

## A target with an ordinary action

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

Return Gameplay Tags for filters from `GetInteractionTags_Implementation`, and local highlighting from `OnFocusStart_Implementation` and `OnFocusEnd_Implementation`.

If a target with the interface can also be taken (it has a grab target component), the server will tell it about pickup and release:

```cpp
// In the class declaration
virtual void OnGrab_Implementation(UInteractorComponent* InInteractor) override;
virtual void OnRelease_Implementation(UInteractorComponent* InInteractor, EInteractionReleaseReason InReason,
                                      FVector InLinearVelocity, FVector InAngularVelocity) override;
```

`OnGrab` arrives when the first hand is accepted, `OnRelease` when the last one lets go; the velocities are already clamped by the server and are zero for any reason other than `Released`.

## Network limits in code

The default limits live in `UInteractionSystemSettings` (**Project Settings → Plugins → Interaction System**). A single interactor overrides a value with a pair "flag + value":

```cpp
// Server and clients must agree; the server's value is final.
Interactor->bOverrideMaxThrowSpeed = true;
Interactor->MaxThrowSpeed = 3200.f;

const float Reach = Interactor->GetMaxOriginDistance(); // effective value: override or project default
const float DefaultTimeout = GetDefault<UInteractionSystemSettings>()->PoseTimeout;
```

For `GetDefault<UInteractionSystemSettings>()` include `InteractionSystemSettings.h`; the `InteractionSystem` module already pulls in the needed `DeveloperSettings`.

## Custom grab target

The ready-made manipulation covers most items. If the motion must be implemented differently (a lever on a hinge, a rope, custom physics), there are two ways: a Blueprint subclass `UInteractionBlueprintGrabTargetComponent` without C++ ([11](11-Blueprint-Integration.md#custom-grab-target-in-blueprint)) or a C++ subclass of `UInteractionGrabTargetComponent`. The contract:

- `CanBeginGrip` is called locally for focus and on the server before a grip — read only replicated state;
- `BeginGrip` is called on the server after reservation; `false` cancels the reservation;
- `EndGrip` removes only this hand; a forced ending (a reason other than `Released`) must not throw the item;
- do not create your own owner registry: that is `UInteractionClaimSubsystem`.

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

Implement the lever's motion yourself, for example in `TickComponent` from `HoldingHand->GetGripTransform()` on the server, replicating the result. To release a hand by force, call on the server `HoldingHand->CancelTarget(GetOwner(), EInteractionReleaseReason::TargetInvalid)`: the interactor will call your `EndGrip` and remove the reservation. For `Use` of a held item override `Use`.

## Server-side manipulation functions

The state-changing functions of `UInteractionManipulationComponent` run only on the server; in Blueprint they are marked authority-only. An example of placing an item on a shelf:

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

| Function | Purpose |
|---|---|
| `AttachToSurface(Parent, Socket)` | Attach a free item to a component of another actor, preserving its world pose. Regrabbing removes the attachment |
| `SetFreeSimulation(bSimulate, Velocity, AngularVelocity)` | Remove the attachment and the external driver and set the simulation, for example to launch an item with a trap |
| `SetExternalDriver(Driver)` / `SetExternalPose(Pose)` | Hand the motion to another actor: a wheel on a door, an arrow on a string |
| `SetGrabEnabled(bEnabled)` | Forbid or allow grabbing; turning it off releases the current hands, the value is replicated |
| `ResetManipulation()` / `CancelAllGrips()` | Return the item home or release all hands without a throw |
| `SetHomeTransform(Transform)` / `GetHomeTransform()` | Change or read the home pose for `ResetManipulation` |
| `SyncTransform()` | Immediately send clients a pose your code changed |
| `ValidateSetup(OutReason)` | Validate the settings; useful in `BeginPlay` with the reason written to the log |

## Subscribing to manipulation events

An example for `AMyRifle`, a subclass of `AInteractionPhysicsActor` (the `Manipulation` member is already in the base class):

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

Dynamic delegate handlers must be `UFUNCTION()`. Do not repeat the `UFUNCTION` names of the base class: `AInteractionPhysicsActor` already has `HandlePhysicsHit`, `HandlePhysicsSleep` and `MulticastImpactSound`, and UHT will not let you declare them again in a subclass.

## Reading the owner on the server

```cpp
#include "InteractionClaimSubsystem.h"

if (const auto* Claims = GetWorld()->GetSubsystem<UInteractionClaimSubsystem>())
{
    const UInteractorComponent* ClaimingHand = Claims->FindClaimant(Item); // first hand or nullptr
    const APawn* OwningPawn = Claims->FindOwningPawn(Item);                // owner of a grip or a linked reservation
    const TArray<AActor*> HeldTargets = Claims->GetClaimedTargets();       // everything held or reserved now
}
```

The ledger is not replicated: on a client use the interactor's `GetHeldActor()` or the manipulation's `GetPrimaryHand()`.

## Math for your own tests

`FInteractionRules` and `FInteractionGripMath` do not touch the world, so they suit quick unit tests of your own profiles:

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
