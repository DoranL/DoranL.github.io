---
layout: single
title: "DEV2 캐릭터 이동 2편"
---

<span style = "color:green">캐릭터 이동2편 내용 정리</span>

***

> Alt - 구르기 
>
> G - 줍기
>
> 왼쪽 마우스 - 기본 공격
>
> 오른쪽 마우스 - 카메라 상하좌우 전환

***

<br/>

이번 장은 캐릭터 이동 마지막 편으로 구르기, 줍기, 기본 공격, 카메라 상하좌우 전환을 

구현하도록 하겠습니다.

TMI이긴 하지만 PELIA 등 뒤에는 애니메이션을 보니까 포탈로 사용 가능한 

원형 물체가 달려있는 데 엔딩쯤에 이 포탈을 사용하여 PELIA가 모시는 신인 

시뮬라크르를 엔딩 창에 모셔오는 것을 구현해 볼 예정입니다.

![이미지](/img/peliaback.JPG)

이번 장에서 다루는 기능들은 비교적 코드가 짧습니다.

<br/>

<mark>구르기</mark>

1. 우선 구르기 몽타주 생성

2. 1편과 동일하게 프로젝트 세팅창에서 입력 항목에 입력 키와 동작 이름을 기입 

3. c++ 코드를 통해 플레이어가 키 입력 시 해당 동작을 수행하기 위해 

   필요한 함수로 이동하도록 바인딩 해주는 과정이 필요

<mark>줍기</mark>

1. 플레이어 캐릭터(PELIA)의 손에 칼을 부착할 수 있는 소켓과 부착하는 칼에 소켓을 추가

2. 프로젝트 세팅창에서 입력 항목에 입력 키와 동작 이름을 기입

3. 무기 콜리전 범위 안에 있고 줍기 키를 입력받았을 때 무기를 주울 수 있도록 범위 설정

4. c++ 코드를 통해 플레이어가 키 입력 시 해당 동작을 수행하기 위해 

   필요한 함수로 이동하도록 바인딩 해주는 과정이 필요

<mark>기본공격</mark>

1. 공격 몽타주 생성(색션 구분)
2. c++ 코드를 통해 공격 몽타주 속도 제어와 구분한 섹션을 동작하도록 함

<mark>카메라 상하좌우 전환</mark>

1. c++ 코드를 통해 카메라 회전의 최소, 최대값을 지정

<br/>



구르기 1.

아래 사진처럼 애니메이션 몽타주를 생성해주고 색션을 구분하여 구르기 애니메이션 종류 후 

블루프린트에서 작성한 것처럼 c++ 코드에서 작성한 stop roll 함수를 호출 가능합니다.

![이미지](/img/rollanim.JPG)

![이미지](/img/roll.JPG)

<center>RollMontage 애니메이션 적용</center>

![이미지](/img/rollblueprint.JPG)

<br/>

구르기 2.

구르기, 줍기, 기본공격, 카메라 상하좌우에 관련된 입력 키 설정 

![이미지](/img/img1_10.JPG)

![이미지](/img/img1_11.JPG)

<br/>

구르기 3.

Roll(헤더파일)

```c++
UPROPERTY(EditAnywhere)
	class UAnimMontage* RollMontage;

	UPROPERTY(VisibleAnywhere, BlueprintReadWrite, Category = "Anims")
	bool bRoll;

	void Roll();
	
	//블루프린트에서 호출할 수 있도록
	UFUNCTION(BlueprintCallable)
	void StopRoll();

	UPROPERTY(VisibleAnywhere, BlueprintReadWrite)
	bool bJump;
```

<br/>



Roll(cpp 파일)

```c++
//구르고 있지 않고 죽지 않았고 W,S 또는 A,D키를 누르고 있을 때 Nelia의 AnimInstance를 가져오고 RollMontage를 1.5배 속도로 실행한다.
void ANelia::Roll()
{
	if (!bRoll && MovementStatus != EMovementStatus::EMS_Death && (bMovingForward || bMovingRight))
	{
		bRoll = true;

		UAnimInstance* AnimInstance = GetMesh()->GetAnimInstance();

		if (RollMontage && AnimInstance)
		{
			AnimInstance->Montage_Play(RollMontage, 1.5f);
			AnimInstance->Montage_JumpToSection(FName("Roll"), RollMontage);
		}
	}
}
```

<br/>

줍기 1.

플레이어 캐릭터(PELIA)의 손에 칼을 부착할 수 있는 소켓과 부착하는 칼에 소켓을 추가

![이미지](/img/handsocket.JPG)

![이미지](/img/weaponsocket.JPG)

<br/>

줍기 2.

구르기 2에서 이미 올렸습니다. 

추가적으로 해당 게임에서 G키와 왼쪽 마우스를 통해 줍기와 기본 공격이 가능하도록 

설정하였기 때문에 Pickup 하나의 액션 매핑에 2개의 키를 넣어주었습니다.

<br/>

줍기 3.

아래 사진에서 노란색 콜리전 안에서 플레이어가 G키 또는 왼쪽 마우스를 클릭했을 경우 플레이어 

오른쪽 손에 부착해둔 RightHandSocket에 무기가 장착되도록 구현되었습니다.

![이미지](/img/collisonweapon.JPG)

<br/>

줍기 4.

줍기 (헤더파일)

```c++
	bool bPickup;
	void PickupPress();
	void PickupReleas();
```

<br/>

줍기 (cpp파일)

```c++
//죽은 상태가 아니고 Item에서 선언한 Collision volume 안에 감지된 아이템이 있고 공격 중이 아니라면
//장착된 아이템을 weapon 형식으로 변환하여 클래스 변수 weapon에 넣어주고 해당 weapon을 장착 이후 SetActiveOverlappingitem에
//들어갔던 아이템들을 모두 비워줌 만약 무기를 이미 장착하고 있으면 combo attack에서 사용되는 saveattack을 true로 하고 
//공격 중이 아닐 경우 Attack() 함수를 호출함
void ANelia::PickupPress()
{
	bPickup = true;

	if (MovementStatus == EMovementStatus::EMS_Death) return;

	if (MainPlayerController) return;

	if (ActiveOverlappingItem && !bAttacking)
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
		if (bAttacking)
		{
			saveAttack = true;
		}
		else
		{
			Attack();
		}
	}
}

void ANelia::PickupReleas()
{
	bPickup = false;
}
```

<br/>

기본공격 1.

rollmontage와 마찬가지로 공격 종료 시점을 EndAttacking을 통해 알려주고 가만히 있는데 적이 칼에 다았다고 데미지가 들어가면 안 되기 때문에 ActivateCollision과 DeactivateCollison을 

사용하여 활성/비활성화를 시켜주었습니다.

![이미지](/img/attackmontage.JPG)

![이미지](/img/attackblueprint.JPG)

<br/>

기본공격 2.

기본공격(헤더파일)

```c++
UPROPERTY(VisibleAnywhere, BlueprintReadWrite, Category = "Anims")
	bool bAttacking;

	UPROPERTY(VisibleAnywhere, BlueprintReadWrite, Category = "Anims")
	bool saveAttack;

	UFUNCTION(BlueprintCallable)
	void ResetCombo();

	UFUNCTION(BlueprintCallable)
	void SaveComboAttack();

	int AttackMotionCount = 0;

	void Attack();

	UFUNCTION(BlueprintCallable)
	void AttackEnd();

	UPROPERTY(EditDefaultsOnly, BlueprintReadOnly, Category = "Anims")
	class UAnimMontage* CombatMontage;
```

<br/>

구현하고 싶은 기본 스킬 동작 방식이  천천히 공격 버튼을 계속해서 누를 경우 1번 동작만 반복을 하고 빠르게 누르면 1,2 또는 1,2,3 동작을 하도록 구현하기 위해 resetcombo와 savecombo를  

사용하였습니다.

```c++
//공격 중인 상태가 아니고 죽지 않았을 때 적 방향을 바라보고 Nelia의 AnimInstance를 가져옴
//case0부터 2까지 순서대로 시행하고 마지막 Attack3를 시행 후 다시 Attack1번부터 공격 모션을 수행함
//CombatMontage에서 공격 애니메이션마다 savecombo와 resetcombo를 지정해 주었고 savecombo와 reset combo 사이에 공격 키를 계속해서 누르면 
//1번, 2번, 3번 기본 공격 동작을 수행하지만 시간을 두고 기본 공격 키 입력 시 계속해서 reset combo에서 공격 모션이 0으로 초기화되기 때문에
//1번 공격만 하도록 구현
void ANelia::Attack()
{
	if (!bAttacking && MovementStatus != EMovementStatus::EMS_Death && EquippedWeapon)
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
				AnimInstance->Montage_Play(CombatMontage, 1.7f);
				AnimInstance->Montage_JumpToSection(FName("Attack"), CombatMontage);
				break;
			case 1:
				AnimInstance->Montage_Play(CombatMontage, 1.7f);
				AnimInstance->Montage_JumpToSection(FName("Attack2"), CombatMontage);
				break;
			case 2:
				AnimInstance->Montage_Play(CombatMontage, 1.1f);
				AnimInstance->Montage_JumpToSection(FName("Attack3"), CombatMontage);
				break;
			default:
				break;
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

void ANelia::ResetCombo()
{
	AttackMotionCount = 0;
	saveAttack = false;
	bAttacking = false;
}

void ANelia::SaveComboAttack()
{
	if (saveAttack)
	{
		saveAttack = false;
		AttackEnd();
		Attack();
	}
}
```

<br/>

위에서 설명했던 것처럼 스킬마다 savecombo와 resetcombo를 간격을 두고 지정해두고 

스킬 사용 이후 다시  키를 누르는 시간에 따라 1, 2, 3번 기본 공격 모션을 하도록 구현

![이미지](/img/saveattack.JPG)

<br/>

카메라 상하좌우 1.

```c++
//CameraBoom은 카메라와 플레이어를 이어주고 있는 것 처럼 보이는 빨간선 부모 컴포넌트인 Nelia의 Capsule Component에 부착되어 있고 그 사이 간격은 400.f
	//bUsePawnControllerRotation은 플레이어를 기준으로 도는 것을 가능하도록 하는 변수 즉 
	//false가 되면 플레이어의 앞 모습을 볼 수 없음 
	CameraBoom = CreateDefaultSubobject<USpringArmComponent>(TEXT("CameraBoom"));   
	CameraBoom->SetupAttachment(GetRootComponent());
	CameraBoom->TargetArmLength = 400.f;							
	CameraBoom->bUsePawnControlRotation = true;						

	//FollowCamera를 카메라붐 끝 부분에 부착. FollowCamera는 이미 CameraBoom이 좌우로 돌기 때문에 		 같이 돌아갈 필요가 없어 false로 둠 단 CameraBoomd bUsePawnControlRotation을 false로두고
	//FollowCamera에서 true로 두면 Nelia를 기준으로 회전하지 않고 카메라를 기준으로 회전하기 때문에 4		캐릭터로부터 400.f 위치에서 카메라 홀로 돌아가게 됨
	FollowCamera = CreateDefaultSubobject<UCameraComponent>(TEXT("FollowCamera"));		 
	FollowCamera->SetupAttachment(CameraBoom, USpringArmComponent::SocketName);
	FollowCamera->bUsePawnControlRotation = false;

	//키 입력 시 1초동안 65씩 회전 BaseTurnRate는 프로젝트 세팅 입력창에서 볼 수 있듯이 
	//좌우 회전 비율이고 BaseLookupRate는 상하 회전 비율이다.
	BaseTurnRate = 65.f;
	BaseLookUpRate = 65.f;

	//카메라와 캐릭터가 동시 회전하는 걸 막는 역할 이렇게 설정하지 않으면 회전 시 캐릭터의 앞 모습을 
	//절대 볼 수 없음.
	//Let that just affect the camera
	bUseControllerRotationYaw = false;
	bUseControllerRotationPitch = false;
	bUseControllerRotationRoll = false;

void ANelia::SetupPlayerInputComponent(UInputComponent* PlayerInputComponent)
{
	Super::SetupPlayerInputComponent(PlayerInputComponent);					
	check(PlayerInputComponent);//입력받은 키를 확인


	PlayerInputComponent->BindAxis("turn", this, &ANelia::Turn);
	PlayerInputComponent->BindAxis("LookUp", this, &ANelia::LookUp);
}

//마우스를 좌우로 움직일 경우 z값이 증가 감소함
void ANelia::Turn(float Value)
{
	if (CanMove(Value))
	{
		AddControllerYawInput(Value);
	}
}

//마우스를 상하로 움직일 경우 y값이 증가 감소함
void ANelia::LookUp(float Value)
{
	if (CanMove(Value))
	{
		AddControllerPitchInput(Value);
	}
}
```

<br/>

PELIA 400만큼 거리 뒤에 Cameraboom을 부착하고 플레이어가 돌면 카메라도 

플레이어를 기준으로 회전하도록 설정

![이미지](/img/cameraboom.JPG)

<br/>



비교적 이번장에서는 기본 공격을 연타하는냐 아니냐에 따라 첫번째 공격모션만 하는지 아니면 

두번째 또는 세번째 공격모션까지 하는지를 설정하는 것이 좀 까다로웠던 부분인 만큼 영상에서 

잘 확인하시면 될 거 같습니다.

<iframe width="560" height="315" src="https://www.youtube.com/embed/VU7UVNvM-Zc" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

다음 DEV3에서는 이 번장에서 다룬 CombatMontage, RollMontage와 추가적으로 Locomotion과 Anim Retaget 방법에 대해 다루겠습니다. 감사합니다.

