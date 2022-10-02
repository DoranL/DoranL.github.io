---
layout: single
title: "Enemy"
---

Enemy 구현

Enemy.cpp

```c++
#include "Enemy.h"
#include "Components/SphereComponent.h"
#include "AIController.h"
#include "Nelia.h"
#include "Kismet/KismetSystemLibrary.h"
#include "Components/CapsuleComponent.h"
#include "Components/BoxComponent.h"
#include "Components/SkeletalMeshComponent.h"
#include "Nelia.h"
#include "Kismet/GameplayStatics.h"
#include "Engine/SkeletalMeshSocket.h"
#include "Sound/SoundCue.h"
#include "Animation/AnimInstance.h"
#include "TimerManager.h"
#include "MainPlayerController.h"

// Sets default values
AEnemy::AEnemy()
{
 	// Set this character to call Tick() every frame.  You can turn this off to improve performance if you don't need it.
	PrimaryActorTick.bCanEverTick = true;

	//AgroSphere은 Rampage의 Capsule Component에 부착되어 있는 자식 컴포넌트로 반지름 600.f는 범위를 나타내며 이 범위 안에 Nelia가 있을 경우 추적을 시작한다.
	AgroSphere = CreateDefaultSubobject<USphereComponent>(TEXT("AgroShere"));
	AgroSphere->SetupAttachment(GetRootComponent());
	AgroSphere->InitSphereRadius(600.f);
	
	/////////////////////////////////////////////////////////
	AgroSphere->SetCollisionResponseToChannel(ECollisionChannel::ECC_WorldDynamic, ECollisionResponse::ECR_Ignore);

	//CombatSphere도 AgroSphere처럼 Capsule Component에 부착되어 있고 반지름은 75.f 이 범위 내 Nelia가 있으면 전투상태가 된다.
	CombatSphere = CreateDefaultSubobject<USphereComponent>(TEXT("CombatSphere"));
	CombatSphere->SetupAttachment(GetRootComponent());
	CombatSphere->InitSphereRadius(75.f);

	///////////////////////각각 왼팔 오른팔에 부착해둔 CombatCollision EnemySocket을 부착해줌 그 위치에 박스 컴포넌트가 생성된다.
	CombatCollision = CreateDefaultSubobject<UBoxComponent>(TEXT("CombatCollision"));
	CombatCollision->AttachToComponent(GetMesh(), FAttachmentTransformRules::SnapToTargetIncludingScale, FName("EnemySocket"));

	CombatCollisionLeft = CreateDefaultSubobject<UBoxComponent>(TEXT("CombatCollisionL"));
	CombatCollisionLeft->AttachToComponent(GetMesh(), FAttachmentTransformRules::SnapToTargetIncludingScale, FName("EnemySocket1"));

	bOverlappingCombatSphere = false;

	Health = 75.f;
	MaxHealth = 100.f;
	Damage = 10.f;

	AttackMinTime = 0.5f;
	AttackMaxTime = 3.5f;

	EnemyMovementStatus = EEnemyMovementStatus::EMS_Idle;

	DeathDelay = 3.f;

	bHasValidTarget = false;
}

// Called when the game starts or when spawned
void AEnemy::BeginPlay()
{
	Super::BeginPlay();

	//Enemy 컨트롤러를 받아 넣어주고 각각 범위 내에 감지될 경우 해당 함수를 호출 ex) CombatCollisionLeft에 감지될 경우 CombatOnOverlapBegin() 호출
	AIController = Cast<AAIController>(GetController());

	AgroSphere->OnComponentBeginOverlap.AddDynamic(this, &AEnemy::AgroSphereOnOverlapBegin);
	AgroSphere->OnComponentEndOverlap.AddDynamic(this, &AEnemy::AgroSphereOnOverlapEnd);

	CombatSphere->OnComponentBeginOverlap.AddDynamic(this, &AEnemy::CombatSphereOnOverlapBegin);
	CombatSphere->OnComponentEndOverlap.AddDynamic(this, &AEnemy::CombatSphereOnOverlapEnd);

	CombatCollision->OnComponentBeginOverlap.AddDynamic(this, &AEnemy::CombatOnOverlapBegin);
	CombatCollision->OnComponentEndOverlap.AddDynamic(this, &AEnemy::CombatOnOverlapEnd);

	CombatCollisionLeft->OnComponentBeginOverlap.AddDynamic(this, &AEnemy::CombatOnOverlapBegin);
	CombatCollisionLeft->OnComponentEndOverlap.AddDynamic(this, &AEnemy::CombatOnOverlapEnd);

	/// <summary>
	/// ////////////////////////////////////////////////////////////////////
	/// </summary>
	CombatCollision->SetCollisionEnabled(ECollisionEnabled::NoCollision);
	CombatCollision->SetCollisionObjectType(ECollisionChannel::ECC_WorldDynamic);
	CombatCollision->SetCollisionResponseToAllChannels(ECollisionResponse::ECR_Ignore);
	CombatCollision->SetCollisionResponseToChannel(ECollisionChannel::ECC_Pawn, ECollisionResponse::ECR_Overlap);

	CombatCollisionLeft->SetCollisionEnabled(ECollisionEnabled::NoCollision);
	CombatCollisionLeft->SetCollisionObjectType(ECollisionChannel::ECC_WorldDynamic);
	CombatCollisionLeft->SetCollisionResponseToAllChannels(ECollisionResponse::ECR_Ignore);
	CombatCollisionLeft->SetCollisionResponseToChannel(ECollisionChannel::ECC_Pawn, ECollisionResponse::ECR_Overlap);

	//몬스터가 플레이어를 때릴 때 카메라에 가려지는 것 막는 코드(Mesh(적 테두리), capsule(내가 설정한 캡슐)이 카메라 시야에 있게되면
	//아래 코드가 없을 시 캐릭터를 보여주기 위해 앞으로 당겨지게 되는데 렉 걸린 것 처럼 보여지게 됨 따라서 무시하도록 코드 작성
	GetMesh()->SetCollisionResponseToChannel(ECollisionChannel::ECC_Camera, ECollisionResponse::ECR_Ignore);
	GetCapsuleComponent()->SetCollisionResponseToChannel(ECollisionChannel::ECC_Camera, ECollisionResponse::ECR_Ignore);
}

// Called every frame
void AEnemy::Tick(float DeltaTime)
{
	Super::Tick(DeltaTime);

}

// Called to bind functionality to input
void AEnemy::SetupPlayerInputComponent(UInputComponent* PlayerInputComponent)
{
	Super::SetupPlayerInputComponent(PlayerInputComponent);
}

//범위 내 검출된 대상이 자기 자신이 아니고 자기 자신이 죽지 않은 상태일 경우 OtherActor가 Nelia인지 확인하고 맞을 시 대상 타겟으로 이동
void AEnemy::AgroSphereOnOverlapBegin(UPrimitiveComponent* OverlappedComponent, AActor* OtherActor, UPrimitiveComponent* OtherComp, int32 OtherBodyIndex, bool bFromSweep, const FHitResult& SweepResult)
{
	if (OtherActor && Alive())
	{
		ANelia* Nelia = Cast<ANelia>(OtherActor);
		if (Nelia)
		{
			MoveToTarget(Nelia);
		}
	}
}

//검출 대상이 자기 자신이 아닌지 확인하고 유효 타겟 변수를 없음으로 두고 Nelia의 CombatTarget이 Enemy일 경우 대상을 nullptr로 초기화 시켜준다.
//만약 Enemy 상태가 idle일 경우 AIController를 멈춘다.
void AEnemy::AgroSphereOnOverlapEnd(UPrimitiveComponent* OverlappedComponent, AActor* OtherActor, UPrimitiveComponent* OtherComp, int32 OtherBodyIndex)
{
	if (OtherActor)
	{
		ANelia* Nelia = Cast<ANelia>(OtherActor);
		if (Nelia)
		{
			bHasValidTarget = false;
			 
			if (Nelia->CombatTarget == this)
			{
				Nelia->SetCombatTarget(nullptr);
				//Nelia->UpdateCombatTarget();
			}

			Nelia->SetHasCombatTarget(false);

			Nelia->UpdateCombatTarget();

			SetEnemyMovementStatus(EEnemyMovementStatus::EMS_Idle);
			if (AIController)
			{
				AIController->StopMovement();
			}
		}
	}
}

//위 if문 내용과 동일 만약 Nelia이면 유효 타겟을 true로 두고
void AEnemy::CombatSphereOnOverlapBegin(UPrimitiveComponent* OverlappedComponent, AActor* OtherActor, UPrimitiveComponent* OtherComp, int32 OtherBodyIndex, bool bFromSweep, const FHitResult& SweepResult)
{
	if (OtherActor && Alive())
	{
		ANelia* Nelia = Cast<ANelia>(OtherActor);
		if (Nelia)
		{
			bHasValidTarget = true;

			Nelia->SetCombatTarget(this);
			Nelia->SetHasCombatTarget(true);
			
			Nelia->UpdateCombatTarget();

			CombatTarget = Nelia;
			bOverlappingCombatSphere = true;
			Attack();
		}
	}
}
void AEnemy::CombatSphereOnOverlapEnd(UPrimitiveComponent* OverlappedComponent, AActor* OtherActor, UPrimitiveComponent* OtherComp, int32 OtherBodyIndex)
{
	if (OtherActor && OtherComp)
	{
		ANelia* Nelia = Cast<ANelia>(OtherActor);
		if (Nelia)
		{
			bOverlappingCombatSphere = false;
			MoveToTarget(Nelia);
			CombatTarget = nullptr;

			if (Nelia->CombatTarget == this)
			{
				Nelia->SetCombatTarget(nullptr);
				Nelia->bHasCombatTarget = false;
				Nelia->UpdateCombatTarget();
			}
			if (Nelia->MainPlayerController)
			{
				USkeletalMeshComponent* NeliaMesh = Cast<USkeletalMeshComponent>(OtherComp);
				if (NeliaMesh) Nelia->MainPlayerController->RemoveEnemyHealthBar();
			}

			GetWorldTimerManager().ClearTimer(AttackTimer);
		}
	}
}
```

Enemy 해당 변수 초기화와 카메라 설정 그리고 추격, 전투 범위를 지정해주는 코드

<br/>

![이미지](/img/img_enemy.JPG)

각각 1번부터 4번까지 오른손, 왼손 콜리전 그리고 작은 원인 3번이 CombatCollsion 4번이 추적 범위인 AgroSphere입니다.

<br/>

```c++
//Nelia에게 이동하도록 타겟으로 잡고 만약 AIController가 있으면 ///////////////////////////////////////////////////////////////////////////////
void AEnemy::MoveToTarget(ANelia* Target)
{
	SetEnemyMovementStatus(EEnemyMovementStatus::EMS_MoveToTarget);

	if (AIController)
	{
		FAIMoveRequest MoveRequest;
		//타겟을 목표 액터로 지정
		MoveRequest.SetGoalActor(Target);
		//이 원 내에 있을 때 타겟으로 인식
		MoveRequest.SetAcceptanceRadius(10.0f);

		FNavPathSharedPtr NavPath;

		AIController->MoveTo(MoveRequest, &NavPath);

	}
}

//해당 OtherActor가 자신이 아니라 Nelia이고 Nelia를 공격 시 다오는 HitParticle이 있을 경우 Enemy에 부착해둔 TipSocket을 불러오고 
// TipSocket이 있으면 소켓이 있는 해당 위치에서 HitParticle을 나타냄 
void AEnemy::CombatOnOverlapBegin(UPrimitiveComponent* OverlappedComponent, AActor* OtherActor, UPrimitiveComponent* OtherComp, int32 OtherBodyIndex, bool bFromSweep, const FHitResult& SweepResult)
{
	if (OtherActor)
	{
		ANelia* Nelia = Cast<ANelia>(OtherActor);
		if (Nelia)
		{
			if (Nelia->HitParticles)
			{
				const USkeletalMeshSocket* TipSocket = GetMesh()->GetSocketByName("TipSocket");
				if (TipSocket)
				{
					FVector SocketLocation = TipSocket->GetSocketLocation(GetMesh());
					UGameplayStatics::SpawnEmitterAtLocation(GetWorld(), Nelia->HitParticles, SocketLocation, FRotator(0.f), false);
				}
			}
		
			if (Nelia->HitSound)
			{
				UGameplayStatics::PlaySound2D(this, Nelia->HitSound);
			}
			/////////
			if (DamageTypeClass)
			{
				UGameplayStatics::ApplyDamage(Nelia, Damage, AIController, this, DamageTypeClass);
			}
		}
	}
}

void AEnemy::CombatOnOverlapEnd(UPrimitiveComponent* OverlappedComponent, AActor* OtherActor, UPrimitiveComponent* OtherComp, int32 OtherBodyIndex)
{

}


//콜리전을 활성화시켜 주는 함수 CombatCollsion, CombatCollisionL은 실제로 내가 설정해준 Enemy의 왼손과 오른손 콜리전 
//만약 콜리전이 활성화된 상태에서 휘두르는 소리가 있을 경우 소리를 재생한다.
void AEnemy::ActivateCollision()
{
	CombatCollision->SetCollisionEnabled(ECollisionEnabled::QueryOnly);
	CombatCollisionLeft->SetCollisionEnabled(ECollisionEnabled::QueryOnly);

	if (SwingSound)
	{
		UGameplayStatics::PlaySound2D(this, SwingSound);
	}
}

//위 내용과 반대의 내용 왼손과 오른손에 있는 콜리전을 NoCollsion을 통해 비활성화 시켜준다.
void AEnemy::DeactivateCollision()
{
	CombatCollision->SetCollisionEnabled(ECollisionEnabled::NoCollision);
	CombatCollisionLeft->SetCollisionEnabled(ECollisionEnabled::NoCollision);
}

//Enemy가 죽지 않았고 유효 타겟이 있을 경우 만약 AIController가 있을 경우 이동을 멈추고 상태를 공격 상태로 지정한다. (이동을 하면서 공격하는 것을 방지하기 위함
//만약 공격 중인 상태가 아니라면 공격 상태를 확인하는 bool 변수를 true로 두고 Enemy의 AnimInstance를 가져오고 Switch 문을 통해 공격 모션을 Attack, Attack1, JumpAttack 순서대로 실행하도록 설정
//한 CombatMontage에 설정해둔 3가지 공격 모션이 조건 만족 시 JumpToSection을 통해 이름이 같은 해당 섹션으로 점프되어 실행된다.
void AEnemy::Attack()
{
	if (Alive() && bHasValidTarget)
	{
		if (AIController)
		{
			AIController->StopMovement();
			SetEnemyMovementStatus(EEnemyMovementStatus::EMS_Attacking);
		}

		if (!bAttacking)
		{
			bAttacking = true;
			UAnimInstance* AnimInstance = GetMesh()->GetAnimInstance();
			if (AnimInstance)
			{
				int32 Section = EnemyAttackCount;
				switch (Section)
				{
				case 0:
					AnimInstance->Montage_Play(CombatMontage, 1.3f);
					AnimInstance->Montage_JumpToSection(FName("Attack"), CombatMontage);
					break;
				case 1:
					AnimInstance->Montage_Play(CombatMontage, 1.f);
					AnimInstance->Montage_JumpToSection(FName("Attack1"), CombatMontage);
					break;
				case 2:
					AnimInstance->Montage_Play(CombatMontage, 1.7f);
					AnimInstance->Montage_JumpToSection(FName("JumpAttack"), CombatMontage);
					break;
				default:
					;
				}
				EnemyAttackCount++;
				if (EnemyAttackCount > 2)
				{
					EnemyAttackCount = 0;
				}
			}
		}
	}
}

//공격 모션 간격을 랜덤으로 설정해준는 부분 play 시 공격 애니메이션 간격이 실행할 때마다 다른 것을 알 수 있다.
void AEnemy::AttackEnd()
{
	bAttacking = false;
	if (bOverlappingCombatSphere)
	{
		float AttackTime = FMath::FRandRange(AttackMinTime, AttackMaxTime);
		GetWorldTimerManager().SetTimer(AttackTimer, this, &AEnemy::Attack, AttackTime);
	}
}


//Enemy 체력 - 공격 받은 데미지량이 0보다 작으면 그 해당 데미지만큼 빼주고 죽은 상태이므로 Nelia의 타겟을 업데이트 해준다.
float AEnemy::TakeDamage(float DamageAmount, struct FDamageEvent const& DamageEvent, class AController* EventInstigator, AActor* DamageCauser)
{
	if (Health - DamageAmount <= 0.f)
	{
		Health -= DamageAmount;
		Die(DamageCauser);
	}
	else
	{
		Health -= DamageAmount;
	}

	return DamageAmount;
}

//상태를 죽은 상태로 두고 DeathMontage에서 애니메이션을 실행 앞에서 설정해둔 Collision과 Capsule Component를 모두 지워준다.
//이후 Nelia의 공격 대상을 업데이트를 하여 새로운 대상을 공격할 수 있도록 해줌.
void AEnemy::Die(AActor* Causer)
{
	SetEnemyMovementStatus(EEnemyMovementStatus::EMS_Dead);
	UAnimInstance* AnimInstance = GetMesh()->GetAnimInstance();
	if (AnimInstance)
	{
		AnimInstance->Montage_Play(CombatMontage, 1.35f);
		AnimInstance->Montage_JumpToSection(FName("Death"), CombatMontage);
	}

	CombatCollision->SetCollisionEnabled(ECollisionEnabled::NoCollision);
	CombatCollisionLeft->SetCollisionEnabled(ECollisionEnabled::NoCollision);
	AgroSphere->SetCollisionEnabled(ECollisionEnabled::NoCollision);
	CombatSphere->SetCollisionEnabled(ECollisionEnabled::NoCollision);

	GetCapsuleComponent()->SetCollisionEnabled(ECollisionEnabled::NoCollision);

	bAttacking = false;

	ANelia* Nelia = Cast<ANelia>(Causer);
	if (Nelia)
	{
		Nelia->UpdateCombatTarget();
	}
}

//죽은 후에는 애니메이션을 멈추고 스켈레톤도 없애준다. //////////// 근데 여기서 타이머는 왜 쓰는거지?
void AEnemy::DeathEnd()
{
	GetMesh()->bPauseAnims = true;
	GetMesh()->bNoSkeletonUpdate = true;

    GetWorldTimerManager().SetTimer(DeathTimer, this, &AEnemy::Disappear, DeathDelay);
}

bool AEnemy::Alive()
{
	return GetEnemyMovementStatus() != EEnemyMovementStatus::EMS_Dead;
}

void AEnemy::Disappear()
{
	Destroy();
}
```

Enemy의 전반적인 애니메이션을 설정하는 코드 

<br/>

![이미지](/img/img_enemy1.JPG)

각각 Attack, Attack1, JumpAttack, Death, JumpFall 몽타주 섹션으로 나누어져 있고 공격 모션에는 노티파이로 콜리전 활성화 시점과 종료 시점을 알리고 공격 종료 시점도 나타내어 애니메이션 상에서 Enemy 공격 모션이 시작하는 시점에서만 데미지를 가할 수 있도록 함

<br/>

![이미지](/img/img_enemy2.JPG)

Enemy_BP 디테일 창에서 보면 기본 Enemy의 기본 상태는 EMS_Idle로 되어 있고 체력과, 최대 체력, 데미지 등을 코드 상에서 BlueprintReadWrite으로 설정해 두었기 때문에 블루프린트에서 보고 수정하는 것이 가능하다. 또한 공격 최소 시간과 최대 시간을 코드 내에서 0.5와 3.5로 지정해 두었는데 이는 적이 공격을 하고 다시 공격을 하는데 걸리는 시간이 최소 0.5초에서 최대 3.5인 것으로 알 수 있다. 

<br/>

Enemy.h

```c++
#pragma once

#include "CoreMinimal.h"
#include "GameFramework/Character.h"
#include "Enemy.generated.h"


UENUM(BlueprintType)
enum class EEnemyMovementStatus :uint8
{
	EMS_Idle				UMETA(DeplayName = "Idle"),
	EMS_MoveToTarget		UMETA(DeplayName = "MoveToTarget"),
	EMS_Attacking			UMETA(DeplayName = "Attacking"),
	EMS_Dead				UMETA(DeplayName = "Dead"),

	EMS_MAX					UMETA(DeplayName = "DefaultMax")
};

UCLASS()
class DREAMCATCHER_API AEnemy : public ACharacter
{
	GENERATED_BODY()

public:
	// Sets default values for this character's properties
	AEnemy();

	//캐릭터가 죽고 나서 계속 공격 당하는 걸 막는 bool 변수
	bool bHasValidTarget;

	UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "Movement")
	EEnemyMovementStatus EnemyMovementStatus;

	FORCEINLINE void SetEnemyMovementStatus(EEnemyMovementStatus Status) { EnemyMovementStatus = Status; }
	FORCEINLINE EEnemyMovementStatus GetEnemyMovementStatus() { return EnemyMovementStatus; }

	UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "AI")
	class USphereComponent* AgroSphere;

	//combatsphere 내에 플레이어가 들어오면 전투
	UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "AI")
	USphereComponent* CombatSphere;

	//왜 AAIController이지
	UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "AI")
	class AAIController* AIController;

	UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "AI")
	float Health;

	UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "AI")
	float MaxHealth;

	UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "AI")
	float Damage;

	UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "AI")
	class UParticleSystem* HitParticles;

	UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "AI")
	class USoundCue* HitSound;

	UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "AI")
	class USoundCue* SwingSound;

	UPROPERTY(VisibleAnywhere, BlueprintReadWrite, Category = "Combat")
	class UBoxComponent* CombatCollision;

	UPROPERTY(VisibleAnywhere, BlueprintReadWrite, Category = "Combat")
	class UBoxComponent* CombatCollisionLeft;

	UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Combat")
	class UAnimMontage* CombatMontage;

	FTimerHandle AttackTimer;

	int EnemyAttackCount = 0;

	UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Combat")
	float AttackMinTime;

	UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Combat")
	float AttackMaxTime;

	UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Combat")
	TSubclassOf<UDamageType> DamageTypeClass;

	FTimerHandle DeathTimer;

	UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "Combat")
		float DeathDelay;

protected:
	// Called when the game starts or when spawned
	virtual void BeginPlay() override;

public:	
	// Called every frame
	virtual void Tick(float DeltaTime) override;

	// Called to bind functionality to input
	virtual void SetupPlayerInputComponent(class UInputComponent* PlayerInputComponent) override;

	UFUNCTION()
	virtual void AgroSphereOnOverlapBegin(UPrimitiveComponent* OverlappedComponent, AActor* OtherActor, UPrimitiveComponent* OtherComp, int32 OtherBodyIndex, bool bFromSweep, const FHitResult& SweepResult);
	UFUNCTION()
	virtual void AgroSphereOnOverlapEnd(UPrimitiveComponent* OverlappedComponent, AActor* OtherActor, UPrimitiveComponent* OtherComp, int32 OtherBodyIndex);

	UFUNCTION()
	virtual void CombatSphereOnOverlapBegin(UPrimitiveComponent* OverlappedComponent, AActor* OtherActor, UPrimitiveComponent* OtherComp, int32 OtherBodyIndex, bool bFromSweep, const FHitResult& SweepResult);
	UFUNCTION()
	virtual void CombatSphereOnOverlapEnd(UPrimitiveComponent* OverlappedComponent, AActor* OtherActor, UPrimitiveComponent* OtherComp, int32 OtherBodyIndex);

	UFUNCTION(BlueprintCallable)
	void MoveToTarget(class ANelia* Target);

	UPROPERTY(VisibleAnywhere, BlueprintReadWrite, Category = "AI")
	bool bOverlappingCombatSphere;

	UPROPERTY(VisibleAnywhere, BlueprintReadWrite, Category = "AI")
	ANelia* CombatTarget;

	UFUNCTION()
	void CombatOnOverlapBegin(UPrimitiveComponent* OverlappedComponent, AActor* OtherActor, UPrimitiveComponent* OtherComp, int32 OtherBodyIndex, bool bFromSweep, const FHitResult& SweepResult);

	UFUNCTION()
	void CombatOnOverlapEnd(UPrimitiveComponent* OverlappedComponent, AActor* OtherActor, UPrimitiveComponent* OtherComp, int32 OtherBodyIndex);

	UFUNCTION(BlueprintCallable)
	void ActivateCollision();

	UFUNCTION(BlueprintCallable)
	void DeactivateCollision();

	UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "Combat")
	bool bAttacking;

	void  Attack();

	UFUNCTION(BlueprintCallable)
	void AttackEnd();

	virtual float TakeDamage(float DamageAmount, struct FDamageEvent const& DamageEvent, class AController* EventInstigator, AActor* DamageCauser) override;

	void Die(AActor* Causer);

	UFUNCTION(BlueprintCallable)
	void DeathEnd();

	bool Alive();

	void Disappear();
};
```

<br/>

Enemy 동작 시현 영상

