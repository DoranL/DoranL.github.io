---
layout: single
title: "DEV1 캐릭터 이동 1편"
---

<span style = "color:red">캐릭터 이동1편 내용 정리</span>

***

> W,A,S,D - 이동
>
> Shift(+ stamina system)  - 달리기 및 스태미나 
>
> spacebar - 점프

***

<br/>

***

***

이번 장에서 움직이도록 할 캐릭터의 이름은 PELIA(Player)이고 스토리 내에서 꿈에 신 

시뮬라크르를 모시는 주요 전사 중 하나입니다. 

![이미지](/img/pelia.JPG)

<br/>

 <mark>캐릭터 이동</mark>

1. 편집 -> 프로젝트 세팅 -> 입력 항목에 입력 키와 동작 이름을 기입해 줘야함 

2. c++ 코드를 통해 플레이어가 키 입력 시 해당 동작을 수행하기 위해 

   필요한 함수로 이동하도록 바인딩 해주는 과정이 필요

3. DEV4에서 다룰 내용인 Blendspace와 Animation blueprint를 사용

<br/>

1.

아래 사진처럼 C++ 코드에서 사용할 동작 이름과 해당 키를 바인딩 시켜줍니다.

![이미지](/img/img1_3.JPG)

![이미지](/img/img1_3_1.JPG)

<br/>

2.

c++ 코드 내에서 SetupPlayerInputComponent 함수를 재정의 하여 사용하기 위해 헤더파일에 아래 사진과 같이 선언을 하고 

![이미지](/img/bind.JPG)

<br/>

cpp 파일에서 부모 클래스 SetupPlayerInputComponent를 상속 받아쓰기위해 super을 사용하여 매개변수 값을 받아 체크하고 프로젝트 세팅창에서 지정해 준 Aaction Name 또는 AxisName을 확인 후 만들어둔 함수로            이동시킵니다.

![이미지](/img/bind1.JPG)

<br/>

<u>주석참고</u>

```c++
//앞,뒤 이동 Value 값을 통해 설정해둔 1,-1 값을 받아 움직인다.
//사용자로부터 W,S키를 입력 받으면 호출되는 함수이고 컨트롤러가 있고 Value가 0이 아니고 
//공격 중인 상태 또는죽은 상태가 아닐 경우 회전값과 방향을 받아 이동한다.
void ANelia::MoveForward(float Value)
{
	bMovingForward = false;

	if (CanMove(Value))
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
	if (CanMove(Value))
	{
		//find out which way is forward
		const FRotator Rotation = Controller->GetControlRotation();
		const FRotator YawRotation(0.f, Rotation.Yaw, 0.f);

		const FVector Direction = FRotationMatrix(YawRotation).GetUnitAxis(EAxis::Y);
		AddMovementInput(Direction, Value);

		bMovingRight = true;
	}
}

//부모 클래스에 있는 점프에 대한 정보를 변경하기에는 문제가 발생할 수 있으므로 헤더에서 점프를 재정의해서 사용
//죽은 상태가 아닐 경우 bool 변수에 true를 넣어줌
void ANelia::Jump()
{
	if (MainPlayerController) if (MainPlayerController->bPauseMenuVisible) return;

	if ((MovementStatus != EMovementStatus::EMS_Death))
	{
		Super::Jump();
		bJump = true;
	} 
}

//spacebar를 누르고 때면 호출되는 함수 bool변수에 false 값을 넣어줌 
void ANelia::StopJumping()
{
	Super::StopJumping();
	bJump = false;
}

//shift 키를 누르는 동안 호출되는 함수 
void ANelia::ShiftKeyDown()
{
	bShiftKeyDown = true;
}

//shift 키를 눌렀다 땠을 때 호출되는 함수 
void ANelia::ShiftKeyUp()
{
	bShiftKeyDown = false;
}
```

<br/>

Stamina system 관련 (헤더파일)

```c++
//블루프린트에서 Nelia 이동 설정할 때 열거형으로 Normal, Sprinting, Dead 중 하나 선택 가능
UENUM(BlueprintType)
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
```

PELIA의 블렌드의 Animation Blueprint이고 해당 이미지는 Idle/Walk/Run스테이트에서 

sprint로 넘어가는 조건을 다루는 부분

![이미지](/img/blueprint.JPG)



<br/>

Stamina system 관련 (cpp파일)

```c++
//시간 당 스테미나 소비량 = 스테미나소비율 * 프레임 
	float DeltaStamina = StaminaDrainRate * DeltaTime;

//스태미나 상태(ESS)에 따른 움직임 상태(EMS) 예를 들어 스태미나가 충분할 시 
//EMS_Sprint로 500.f의 속도로 달리고 부족할 시 EMS_Normal 기본 달리기 300.f의 속도로 이동

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

 스태미나 상태에 따른 HUD 변화는 DEV5에서 다루도록 하겠습니다.

<br/>



자막과 음성이 없고 해상도가 1080p이 최대라서 그런지 조금 깨지는 게 아쉽지만

이번 DEV1에서 다룬 것들을 영상으로 올리니까 위에 빨간색 글씨로 적힌 

<span style = "color:red">캐릭터 이동1편 내용 정리</span> 부분을 참고하며 영상을 보시면 될 거 같습니다.



기존 다운로드한 맵에 보스 몬스터를 넣을 장소가 마땅치 않아서 아쉬운 대로 보스 몬스터 격투장 만들었는데 

시간이 남으면 좀 더 제대로 만들어 보겠습니다.

<iframe width="560" height="315" src="https://www.youtube.com/embed/hf0laVC8vcQ" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

***

***

다음 DEV2에서는 캐릭터 이동 2편으로 구르기, 줍기, 기본 공격(왼쪽 마우스), 

카메라 전환(오른쪽 마우스)을 추가적으로 다루겠습니다.  

가능하면 DEV5에서 다루기로 한 HUD도 추가하여 블로그를 작성하도록 하겠습니다.