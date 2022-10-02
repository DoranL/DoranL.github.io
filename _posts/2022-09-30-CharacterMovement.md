---
layout: single
title: "Character"
---

메인 캐릭터의 이동, 애니메이션, UI 등을 구현해보자

Nelia.cpp - 

```c++
#include "Nelia.h"
#include "GameFramework/SpringArmComponent.h"
#include "Camera/CameraComponent.h"
#include "GameFramework/PlayerController.h"
#include "Kismet/GameplayStatics.h"
#include "Engine/World.h"
#include "Components/CapsuleComponent.h"
#include "GameFramework/CharacterMovementComponent.h"
#include "Kismet/KismetSystemLibrary.h"
#include "Weapon.h"
#include "Components/SkeletalMeshComponent.h"
#include "Components/SkeletalMeshComponent.h"
#include "Animation/AnimInstance.h"
#include "Enemy.h"
#include "MainAnimInstance.h"
#include "Sound/SoundCue.h"
#include "Kismet/KismetMathLibrary.h"
#include "MainPlayerController.h"
#include "NeliaSaveGame.h"

// Sets default values
ANelia::ANelia()
{
 	// Set this character to call Tick() every frame.  You can turn this off to improve performance if you don't need it.
	PrimaryActorTick.bCanEverTick = true;

	//CameraBoom은 카메라와 플레이어를 이어주고 있는 것 처럼 보이는 빨간선 부모 컴포넌트인 Nelia의 Capsule Component에 부착되어 있고 그 사이 간격은 400.f
	//bUsePawnControllerRotation은 플레이어를 기준으로 도는 것을 가능하도록 하는 변수
	CameraBoom = CreateDefaultSubobject<USpringArmComponent>(TEXT("CameraBoom"));   
	CameraBoom->SetupAttachment(GetRootComponent());
	CameraBoom->TargetArmLength = 400.f;							
	CameraBoom->bUsePawnControlRotation = true;						

	//Nelia의 캡슐 크기를 c++울 통해 조정 가능
	GetCapsuleComponent()->SetCapsuleSize(20.f, 76.f);				


	//FollowCamera를 카메라붐 끝 부분에 부착. FollowCamera는 이미 CameraBoom이 좌우로 돌기 때문에 같이 돌아갈 필요가 없어 false로 둠 단 CameraBoomd bUsePawnControlRotation을 false로두고
	//FollowCamera에서 true로 두면 Nelia를 기준으로 회전하지 않고 카메라를 기준으로 회전하기 때문에 캐릭터로부터 400.f 위치에서 홀로 돌아가게 됨
	FollowCamera = CreateDefaultSubobject<UCameraComponent>(TEXT("FollowCamera"));		 
	FollowCamera->SetupAttachment(CameraBoom, USpringArmComponent::SocketName);
	FollowCamera->bUsePawnControlRotation = false;

	//키 입력 시 1초동안 65씩 회전 BaseTurnRate는 프로젝트 세팅 입력창에서 볼 수 있듯이 좌우 회전 비율이고 BaseLookupRate는 상하 회전 비율이다.
	BaseTurnRate = 65.f;
	BaseLookUpRate = 65.f;

	//카메라와 캐릭터가 동시 회전하는 걸 막는 역할 이렇게 설정하지 않으면 회전 시 캐릭터의 앞 모습을 절대 볼 수 없음.
	//Let that just affect the camera
	bUseControllerRotationYaw = false;
	bUseControllerRotationPitch = false;
	bUseControllerRotationRoll = false;

	GetCharacterMovement()->bOrientRotationToMovement = true; //bOrientRotationToMovement는 자동으로 캐릭터의 이동방향에 맞춰, 회전 보간을 해준다.
	GetCharacterMovement()->RotationRate = FRotator(0.0f, 540.f, 0.0f); //키 입력 시 540씩 회전함
	GetCharacterMovement()->JumpZVelocity = 650.f; //점프하는 힘
	GetCharacterMovement()->AirControl = 0.2f; //중력의 힘

	MaxHealth = 100.f;
	Health = 65.f;
	MaxStamina = 150.f;
	Stamina = 120.f;

	Speed = 650.f;
	SprintingSpeed = 950.f;

	bShiftKeyDown = false;
	bPickup = false;

	//열거형 EMovementStatus와 EStaminaStatus를 ESS_Normal로 초기화 시켜줌
	MovementStatus = EMovementStatus::EMS_Normal;
	StaminaStatus = EStaminaStatus::ESS_Normal;

	//StaminaDrainRate는 시간당 스테미나 소비량인 DeltaStamina 수식에 사용 DeltaStamina = StaminaDrainRate * DeltaTime;
	//MinSprintStamina는 아래 스위치 문에서 최소 스프린트 수치 이하일 경우 BelowMinimum(최소 이하) 상태로 변환하고 이 상태에서 스프린터키를 
	//누르게 되면 Exhausted 상태로 변화하여 달릴 수 없게됨
	StaminaDrainRate = 25.f;
	MinSprintStamina = 50.f;	

	InterpSpeed = 15.f;
	bInterpToEnemy = false;

	bMovingForward = false;
	bMovingRight = false;

	bHasCombatTarget = false;

	bRoll = false;
}

//게임 플레이 시 재정의 되는 부분
void ANelia::BeginPlay()
{
	Super::BeginPlay();

	MainPlayerController = Cast<AMainPlayerController>(GetController());
}
 
//부모 클래스에 있는 점프에 대한 정보를 변경하기에는 문제가 발생할 수 있으므로 헤더에서 점프를 override해서 다른 것은 부모의 점프를 상속 받고 bJump는 부모가 아닌 Nelia에 정의하여 사용하는 것
//if(MovementStatus != EMovementStatus::EMS_Death)는 공격 받은 이후 죽었을 때도 점프가 가능한 상황을 막는 코드 
void ANelia::Jump()
{
	if (MovementStatus != EMovementStatus::EMS_Death)
	{
		Super::Jump();
		bJump = true;
	} 
}

void ANelia::StopJumping()
{
	Super::StopJumping();
	bJump = false;
}
```

Nelia 카메라와 설정과 변수 초기화 부분

<br/>

![이미지](/img/img1_1.JPG)

이미지를 보면 Nelia의 Capsule Component의 자식으로 Camera Boom 그 자식으로 Follow Camera가 있는 것을 볼 수 있다.

<br/>

***

Nelia.cpp

Nelia의 Stamina 상태에 따른 움직임 지정하는 부분 Left Shift를 누를 시 Tick 함수 내부에 있기 때문에 매 프레임마다 스태미나가 감소되고 버튼을 안 누르면 증가된다.  Max Stamina를 150.f로 해두었기 때문에 150까지 증가한다.

```c++
void ANelia::Tick(float DeltaTime)
{
	Super::Tick(DeltaTime);


	if (MovementStatus == EMovementStatus::EMS_Death) return;


	//시간 당 스테미나 소비량 = 스테미나소비율 * 프레임 
	float DeltaStamina = StaminaDrainRate * DeltaTime;

	//스태미나 상태(ESS)에 따른 움직임 상태(EMS) 예를 들어 스태미나가 충분할 시 EMS_Sprint로 950.f의 속도로 달리고 부족할 시 EMS_Normal 기본 달리기 650.f의 속도로 이동
	switch (StaminaStatus)
	{
	
	case EStaminaStatus::ESS_Normal:
		if (bShiftKeyDown)
		{
			if (Stamina - DeltaStamina <= MinSprintStamina)
			{
				SetStaminaStatus(EStaminaStatus::ESS_BelowMinimum);
				Stamina -= DeltaStamina;
			}
			else
			{
				Stamina -= DeltaStamina;
			}
			SetMovementStatus(EMovementStatus::EMS_Sprinting);
			if (bMovingForward || bMovingRight)
			{
				SetMovementStatus(EMovementStatus::EMS_Sprinting);
			}
			else
			{
				SetMovementStatus(EMovementStatus::EMS_Normal);
			}
		}

		else //shift key up
		{
			if (Stamina + DeltaStamina >= MaxStamina)
			{
				Stamina = MaxStamina;
			}
			else
			{
				Stamina += DeltaStamina;
			}
			SetMovementStatus(EMovementStatus::EMS_Normal);
		}
		break;

	case EStaminaStatus::ESS_BelowMinimum:
		if (bShiftKeyDown)
		{
			if (Stamina - DeltaStamina <= 0.f)
			{
				SetStaminaStatus(EStaminaStatus::ESS_Exhausted);
				Stamina = 0;
				SetMovementStatus(EMovementStatus::EMS_Normal);
			}
			else
			{
				Stamina -= DeltaStamina;
				SetMovementStatus(EMovementStatus::EMS_Sprinting);
			}
		}
		else //shift key up
		{
			if (Stamina + DeltaStamina >= MinSprintStamina)
			{
				SetStaminaStatus(EStaminaStatus::ESS_Normal);
				Stamina += DeltaStamina;
			}
			else 
			{
				Stamina += DeltaStamina;
			}
			SetMovementStatus(EMovementStatus::EMS_Normal);
		}
		break;

	case EStaminaStatus::ESS_Exhausted:
		if (bShiftKeyDown)
		{
			Stamina = 0.f;
		}
		else //shift key up
		{
			SetStaminaStatus(EStaminaStatus::ESS_ExhaustedRecovering);
			Stamina += DeltaStamina;
		}
		SetMovementStatus(EMovementStatus::EMS_Normal);
		break;

	case EStaminaStatus::ESS_ExhaustedRecovering:
		if (Stamina + DeltaStamina >= MinSprintStamina)
		{
			SetStaminaStatus(EStaminaStatus::ESS_Normal);
			Stamina += DeltaStamina;
		}
		else
		{
			Stamina += DeltaStamina;
		}
		SetMovementStatus(EMovementStatus::EMS_Normal);
		break;

	default:
		;
	}	
```

<br/>

![이미지](/img/img_stamina.JPG)

![이미지](/img/img_stamina2.JPG)

![이미지](/img/img_stamina1.JPG)

가려져서 잘 보이지는 않지만 좌측 상단에 3가지 stamina 상태를 볼 수 있다.

<br/>

![이미지](/img/img_staminaHUD.JPG)

왼쪽 빨강색 박스가 체력 파랑색 박스가 스태미나 박스이고 우측 박스 슬롯을 통해 체력,스태미나 박스 위치,  크기를 지정해줍니다.

<br/>

![이미지](/img/img_staminaBP.JPG)

이미지에서 볼 수 있듯이 스태미나가 Normal, BelowMinimum, Exhausted, Exhausted Recovering 상태에 따라 스태미나바 색상을 다르게 설정해주었습니다.

<br/>

***

Nelia.cpp 

전투 시 적 방향을 바라보게 하는 보간 기능을 수행하는 코드

```c++
	//적을 바라보고 있고 전투대상이 있으면 실행되고 
	//LookAtYaw = 전투 대상의 위치를 받고 전투 대상과 플레이어의 회전 보간은 InterpRotation = 플레이어의 회전 정도, 전투대상 위치, 프레임당 , 보간 속도)를 
	//받아 SetActorRotation(InterpRotation)을 사용하여 보간을 한다. 즉 전투 대상의 위치를 받고 대상을 바라 보기 위한 회전 값을 받아서 SetActorRotation(InterpRotation)을 통해 보간한다.
	if (bInterpToEnemy && CombatTarget)
	{
		FRotator LookAtYaw = GetLookAtRotationYaw(CombatTarget->GetActorLocation());
		FRotator InterpRotation = FMath::RInterpTo(GetActorRotation(), LookAtYaw, DeltaTime, InterpSpeed);


		SetActorRotation(InterpRotation);
	}
	//////////////////////////////////////////////////////////////////////////////////////////////////////////////////
	if (CombatTarget)
	{
		CombatTargetLocation = CombatTarget->GetActorLocation();
		if (MainPlayerController)
		{
			MainPlayerController->EnemyLocation = CombatTargetLocation;
		}
	}
}

FRotator ANelia::GetLookAtRotationYaw(FVector Target)
{
	FRotator LookAtRotation = UKismetMathLibrary::FindLookAtRotation(GetActorLocation(), Target);
	FRotator LookAtRotationYaw(0.f, LookAtRotation.Yaw, 0.f);
	return LookAtRotationYaw;
}
```

<br/>

***

Nelia.cpp 

플레이어에게 입력 키를 받아 각 함수를 호출함

```c++
// Called to bind functionality to input
void ANelia::SetupPlayerInputComponent(UInputComponent* PlayerInputComponent)
{
	Super::SetupPlayerInputComponent(PlayerInputComponent);					
	check(PlayerInputComponent);											//입력받은 키를 확인


	//입력 받은 키에 따라 해당 이름에 맞는 함수 호출? JUMP, 이동, 캐릭터 회전등 
	PlayerInputComponent->BindAction("Jump", IE_Pressed, this, &ANelia::Jump);				
	PlayerInputComponent->BindAction("Jump", IE_Released, this, &ANelia::StopJumping);

	PlayerInputComponent->BindAction("Sprint", IE_Pressed, this, &ANelia::ShiftKeyDown);
	PlayerInputComponent->BindAction("Sprint", IE_Released, this, &ANelia::ShiftKeyUp);

	PlayerInputComponent->BindAction("Roll", IE_Pressed, this, &ANelia::Roll);

	PlayerInputComponent->BindAction("Pickup", IE_Pressed, this, &ANelia::PickupPress);
	PlayerInputComponent->BindAction("Pickup", IE_Released, this, &ANelia::PickupReleas);

	PlayerInputComponent->BindAxis("MoveForward", this, &ANelia::MoveForward);
	PlayerInputComponent->BindAxis("MoveRight", this, &ANelia::MoveRight);

	PlayerInputComponent->BindAxis("turn", this, &APawn::AddControllerYawInput);
	PlayerInputComponent->BindAxis("LookUp", this, &APawn::AddControllerPitchInput);
	PlayerInputComponent->BindAxis("TurnRate", this, &ANelia::TurnAtRate);
	PlayerInputComponent->BindAxis("LokkUpRate", this, &ANelia::LookUpAtRate);
}


//앞,뒤 이동 Value 값을 통해 설정해둔 1,-1 값을 받아 움직인다.
//사용자로부터 W,S키를 입력 받으면 호출되는 함수이고 컨트롤러가 있고 Value가 0이 아니고 공격, 죽은 상태가 아닐 경우 회전값과 방향을 받아 이동한다.
void ANelia::MoveForward(float Value)
{
	bMovingForward = false;
	if ((Controller != nullptr) && (Value != 0.0f) && (!bAttacking) &&(MovementStatus!=EMovementStatus::EMS_Death))
	{
		const FRotator Rotation = Controller->GetControlRotation();
		const FRotator YawRotation(0.f, Rotation.Yaw, 0.f);

		const FVector Direction = FRotationMatrix(YawRotation).GetUnitAxis(EAxis::X);
		AddMovementInput(Direction, Value);

		bMovingForward = true;
	}
}

//좌,우 이동 위 MoveForward랑 구현 방식이 같음
void ANelia::MoveRight(float Value)
{
	bMovingRight = false;
	if ((Controller != nullptr) && (Value != 0.0f) && (!bAttacking) && (MovementStatus != EMovementStatus::EMS_Death))
	{
		//find out which way is forward
		const FRotator Rotation = Controller->GetControlRotation();
		const FRotator YawRotation(0.f, Rotation.Yaw, 0.f);

		const FVector Direction = FRotationMatrix(YawRotation).GetUnitAxis(EAxis::Y);
		AddMovementInput(Direction, Value);

		bMovingRight = true;
	}
}

//키를 누르고 있으면 컨트롤러가 1초안에 65도 회전 가능  
void ANelia::TurnAtRate(float Rate)
{
	AddControllerYawInput(Rate * BaseTurnRate * GetWorld()->GetDeltaSeconds());
}

void ANelia::LookUpAtRate(float Rate)
{
	AddControllerPitchInput(Rate * BaseLookUpRate * GetWorld()->GetDeltaSeconds());
}

//줍기 죽은 상태가 아니고 Item 클래스 자식인 Weapon 클래스에서 장착한 대상이 Nelia일 경우 해당 무기를 Weapon형태로 변환하고 장착 이후 nullptr로 값을 비워준다.
void ANelia::PickupPress()
{
	bPickup = true;

	if (MovementStatus == EMovementStatus::EMS_Death) return;

	if (ActiveOverlappingItem)
	{
		AWeapon* Weapon = Cast<AWeapon>(ActiveOverlappingItem);
		if (Weapon)
		{
			Weapon->Equip(this);
			SetActiveOverlappingItem(nullptr);
		}
	}
	else if (EquippedWeapon)
	{
		Attack();
	}
}

void ANelia::PickupReleas()
{
	bPickup = false;
}

void ANelia::ShiftKeyDown()
{
	bShiftKeyDown = true;
}

void ANelia::ShiftKeyUp()
{
	bShiftKeyDown = false;
}
```

<br/>

![이미지](/img/img1_3.JPG)

프로젝트세팅 입력창에서 설정해준 액션 매핑과 축 매핑 W,S키처럼 Scalse을 1.0과 -1.0으로 설정해두고 코드에서 Value 값을 통해 값을 전달하여 이동한다.

***



Nelia.cpp

```c++
//CombatMontage를 1.2배 속도로 애니메이션을 실행하고 CombatMontage의 몽타주 섹션 Death 부분으로 이동
void ANelia::Die()
{
	if (MovementStatus == EMovementStatus::EMS_Death) return;
	UAnimInstance* AnimInstance = GetMesh()->GetAnimInstance();
	if (AnimInstance && CombatMontage)
	{
		AnimInstance->Montage_Play(CombatMontage, 1.2f);
		AnimInstance->Montage_JumpToSection(FName("Death"));
	}
	SetMovementStatus(EMovementStatus::EMS_Death);
}

//죽으면 애니메이션을 멈추고 스켈레톤 업데이트도 멈춘다.
void ANelia::DeathEnd()
{
	GetMesh()->bPauseAnims = true;
	GetMesh()->bNoSkeletonUpdate = true;
}

//위 Switch문에서 EMS_Sprinting 상태일 때는 속도를 SprintingSpeed=950.f로 값을 주고 그 외에는 Speed인 650.f
void ANelia::SetMovementStatus(EMovementStatus Status)
{
	MovementStatus = Status;
	if (MovementStatus == EMovementStatus::EMS_Sprinting)
	{
		GetCharacterMovement()->MaxWalkSpeed = SprintingSpeed;
	}
	else
	{
		GetCharacterMovement()->MaxWalkSpeed = Speed;
	}
}

//공격 중인 상태가 아니고 죽지 않았을 때 적 방향을 바라보고 Nelia의 AnimInstance를 가져옴
//case0부터 2까지 순서대로 시행하고 마지막 Attack3를 시행 후 다시 Attack1번부터 공격 모션을 수행함
void ANelia::Attack()
{
	if (!bAttacking && MovementStatus != EMovementStatus::EMS_Death)
	{
		bAttacking = true;
		SetInterpToEnemy(true);

		UAnimInstance* AnimInstance = GetMesh()->GetAnimInstance();
		if (AnimInstance && CombatMontage)
		{
			int32 Section = AttackMotionCount;
			switch (Section)
			{
			case 0:
				AnimInstance->Montage_Play(CombatMontage, 2.4f);
				AnimInstance->Montage_JumpToSection(FName("Attack1"), CombatMontage);
				break;
			case 1:
				AnimInstance->Montage_Play(CombatMontage, 2.3f);
				AnimInstance->Montage_JumpToSection(FName("Attack2"), CombatMontage);
				break;
			case 2:
				AnimInstance->Montage_Play(CombatMontage, 1.5f);
				AnimInstance->Montage_JumpToSection(FName("Attack3"), CombatMontage);
			default:
				;
			}
		}
		AttackMotionCount++;
		if (AttackMotionCount > 2)
		{
			AttackMotionCount = 0;
		}
	}
}

void ANelia::AttackEnd()
{
	bAttacking = false;
	SetInterpToEnemy(false);
	if (bPickup)
	{
		Attack();
	}
}
```

<br/>

![이미지](/img/img_NeliaLocomotion.JPG)

블루프린트를 통해 sprint, jump, 무장을 한 상태의 스프린트와 달리기 등을 구현함

<br/>

![이미지](/img/img_combatmontage.JPG)

![이미지](/img/img_NeliaAnim.JPG)

Montage를 만들어 공격 모션 3가지를 넣고 각각 칼을 휘두를 타이밍에 맞춰 시작 타이밍에 ActivateCollision을 끝나는 타이밍에는 DeactivateCollision을 넣어줘 칼에 장착해둔 콜리전을 활성화 비활성화 시켜줬다.

그리고 공격, 그리고 죽음이 끝나는 시점에는 EndAttacking, DeathEnd를 넣어 애니메이션이 끝났음을 알렸다.

***



<br/>

Nelia.h

동작 상태와 스태미나 상태를 열거형으로 선언하여 블루프린터에서 보면 드롭다운 형식으로 선택 가능한 것을 볼 수 있음

```c++
// Fill out your copyright notice in the Description page of Project Settings.

#pragma once

#include "CoreMinimal.h"                   //언리얼 오브젝트가 동작할 수 있는 최소 기능만 선언된 헤더 파일 ex)Templates , Generic, Containers, Math 포함
#include "GameFramework/Character.h"
#include "Nelia.generated.h"

UENUM(BlueprintType)					  //블루프린트에서 Nelia 이동 설정할 때 열거형으로 Normal, Sprinting, Dead 중 하나 선택 가능
enum class EMovementStatus : uint8        //enum클래스 열거형 
{
	EMS_Normal		 UMETA(DisplayName = "Normal"), 
	EMS_Sprinting	 UMETA(DisplayName = "Sprinting"),
	EMS_Death		 UMETA(DisplayName = "Dead"),

	EMS_MAX UMETA(DisplayName = "DefaultMAX")
};

UENUM(BlueprintType)						
enum class EStaminaStatus : uint8
{
	ESS_Normal				UMETA(DisplayName = "Normal"),
	ESS_BelowMinimum		UMETA(DisplayName = "BelowMinimum"),
	ESS_Exhausted		    UMETA(DisplayName = "Exhausted"),
	ESS_ExhaustedRecovering UMETA(DisplayName = "ExhaustedRecovering"),

	ESS_MAX UMETA(DisplayName = "DefaultMax")
};

UCLASS()
class DREAMCATCHER_API ANelia : public ACharacter
{
	GENERATED_BODY()

public:
	// Sets default values for this character's properties
	ANelia();

	UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Combat")
	bool bHasCombatTarget;

	FORCEINLINE void SetHasCombatTarget(bool HasTarget) { bHasCombatTarget = HasTarget; }

	UPROPERTY(BlueprintReadWrite, VisibleAnywhere, Category = "Combat")
	FVector CombatTargetLocation;

	UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "Controller")
	class AMainPlayerController* MainPlayerController;

	//속성 창에서 보여지나 편집이 불가능하고 블루프린터에서 읽기쓰기가 가능한 위에서 선언한 enum 클래스형 EMovementStatus형 MovementStatus 변수 생성 
	UPROPERTY(VisibleAnywhere, BlueprintReadWrite, Category = "Enums")
	EMovementStatus MovementStatus;

	//속성 창에서 보여지나 편집이 불가능하고 블루프린터에서 읽기쓰기가 가능한 위에서 선언한 enum 클래스형 EStaminaSatus형 StaminaStatus 변수 생성
	UPROPERTY(VisibleAnywhere, BlueprintReadWrite, Category = "Enums")
	EStaminaStatus StaminaStatus;

	//인라인 함수를 강제로 수행하는 FORCEINLINE을 사용하여 /////////////// 함수호출 부분의 내용을 가져온다>??????//
	FORCEINLINE void SetStaminaStatus(EStaminaStatus Status) { StaminaStatus = Status; }

	//속성 창에서 편집이 가능하고 블루프린터에서 읽기쓰기가 모두 가능한 float형 변수인 StaminDrainRate 생성
	UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Movement")
	float StaminaDrainRate;

	UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Movement")
	float MinSprintStamina;

	float InterpSpeed;
	bool bInterpToEnemy;
	void SetInterpToEnemy(bool Interp);

	UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "Combat")
	class AEnemy* CombatTarget;

	FORCEINLINE void SetCombatTarget(AEnemy* Target) { CombatTarget = Target; }

	FRotator GetLookAtRotationYaw(FVector Target);

	/** Set Movement status and running speed */
	void SetMovementStatus(EMovementStatus Status);

	UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Running")
	float Speed;

	UPROPERTY(EditAnywhere, BlueprintReadOnly, Category = "Running")
	float SprintingSpeed;

	bool bShiftKeyDown;

	/** Pressed down to enable sprinting */
	void ShiftKeyDown();

	/** Released to stop sprinting */
	void ShiftKeyUp();

	/**Camera boom positioning the camera behind the player*/
	UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = Camera, meta = (AllowPrivateAcess = "true"))
	class USpringArmComponent* CameraBoom;

	/** Follow Camera */
	UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = Camera, meta = (AllowPrivateAcess = "true"))
	class UCameraComponent* FollowCamera;

	/** Base turn rates to scale turning functions for the camera */
	UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = Camera)
	float BaseTurnRate;

	UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = Camera)
	float BaseLookUpRate;


	//Player Stats
	UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Player Stats")
	float MaxHealth;

	UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Player Stats")
	float Health;

	UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Player Stats")
	float MaxStamina;

	UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Player Stats")
	float Stamina;

	virtual float TakeDamage(float DamageAmount, struct FDamageEvent const& DamageEvent, class AController* EventInstigator, AActor* DamageCauser) override;

	void Die(); 

protected:
	// Called when the game starts or when spawned
	virtual void BeginPlay() override;

	virtual void Jump() override;
	virtual void StopJumping() override;

public:	
	// Called every frame
	virtual void Tick(float DeltaTime) override;

	// Called to bind functionality to input
	virtual void SetupPlayerInputComponent(class UInputComponent* PlayerInputComponent) override;

	/** called for forwards/backwards input*/
	void MoveForward(float Value);

	/** called for side to side input*/
	void MoveRight(float Value);

	bool bMovingForward;
	bool bMovingRight;

	/** Called via input to turn at a given rate
	* @parm Rate This is a normalized rate, i.e 1.0 means 100% of desired turn rate
	*/
	void TurnAtRate(float Rate);

	/** Called via input to look up/down at a given rate
	* @parm Rate This is a normalized rate, i.e 1.0 means 100% of desired look up/down rate
	*/
	void LookUpAtRate(float Rate);

	bool bPickup;
	void PickupPress();
	void PickupReleas();

	FORCEINLINE class USpringArmComponent* GetCameraBoom() const { return CameraBoom; }
	FORCEINLINE class UCameraComponent* GetFollowCamera() const { return FollowCamera; }

	UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = Items)
	class AWeapon* EquippedWeapon;

	UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = Items)
    class AItem* ActiveOverlappingItem;

	//장착 무기를 설정 매개변수로 무기를 가지고 (EquippedWeapon = WeaponToSet)	
	FORCEINLINE void SetEquippedWeapon(AWeapon* WeaponToSet) { EquippedWeapon = WeaponToSet; }

	//FORCEINLINE AWeapon* GetEquippedWeapon() { return EquippedWeapon; }
	FORCEINLINE void SetActiveOverlappingItem(AItem* Item) { ActiveOverlappingItem = Item; }

	UPROPERTY(VisibleAnywhere, BlueprintReadWrite, Category = "Anims")
	bool bAttacking;

	int AttackMotionCount = 0;

	void Attack();

	UFUNCTION(BlueprintCallable)
	void AttackEnd();

	UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Anims")
	class UAnimMontage* CombatMontage;

	
	/// ////////////////////
	UPROPERTY(EditAnywhere)
	class UAnimMontage* RollMontage;

	UPROPERTY(VisibleAnywhere, BlueprintReadWrite, Category = "Anims")
	bool bRoll;

	void Roll();

	UFUNCTION(BlueprintCallable)
	void StopRoll();

	UPROPERTY(VisibleAnywhere, BlueprintReadWrite)
	bool bJump;

	class UMainAnimInstance* MainAnimInstance;

	UFUNCTION(BlueprintCallable)
	void DeathEnd();

	void UpdateCombatTarget();

	UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Combat")
	TSubclassOf<AEnemy> EnemyFilter;
};
```

***



<br/>



캐릭터 구현 유투브 영상

<iframe width="560" height="315" src="https://www.youtube.com/embed/KqW8_XflI3o" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

