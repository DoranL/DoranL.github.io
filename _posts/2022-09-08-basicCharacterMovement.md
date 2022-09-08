---
layout: single
title: "Dev1 Basic Character Movement"
---

1. Camera 위치 지정 및 카메라 회전

```c++
ANelia::ANelia()
{
	//Create Camera boom(충격이 있는 경우 플레이어를 향해 당김)
	CameraBoom = CreateDefaultSubobject<USpringArmComponent>(TEXT("CameraBoom"));   
	CameraBoom->SetupAttachment(GetRootComponent());
	CameraBoom->TargetArmLength = 600.f;	//카메라가 Neila와 400떨어진 위치를 고정으로 따라옴
	CameraBoom->bUsePawnControlRotation = true;			//컨트롤러에 따라 CameraArm 회전

	
	GetCapsuleComponent()->SetCapsuleSize(17.f, 64.f);	//Nelia의 캡슐 높이와 반지름 지정

	FollowCamera = CreateDefaultSubobject<UCameraComponent>(TEXT("FollowCamera"));		 
	FollowCamera->SetupAttachment(CameraBoom, USpringArmComponent::SocketName);
	//Attach the camera to the end of the boom and let the boom adjust to match
	//the controller orientation
	FollowCamera->bUsePawnControlRotation = false;	//컨트롤러에 따라 CameraArm 회전하지 않는다

	//Don't rotate when the controller rotates //카메라와 캐릭터가 동시 회전하는 걸 막는 역할
	//Let that just affect the camera
	bUseControllerRotationYaw = false;
	bUseControllerRotationPitch = false;
	bUseControllerRotationRoll = false;
}
```

CameraBoom은 카메라를 의미하며 CreateDefaultSubobject는 생성자에서만 실행 가능하며 탬플릿으로 <>안에 사용한 컴포넌트를 넣어주면 된다.

Text()는 각각의 컴포넌트를 구분하기 위한 값이 들어가며 해쉬 값을 생성하기 때문에, 다른 TEXT값이 겹치면 안된다.

SetupAttachment() 함수는 인자 하위에 위치하게 만든다.

![이미지](/img/img1_1.JPG)

<u>즉 CameraBoom은 컨트롤러에 따라  CameraArm을 회전시키고 캐릭터 Nelia와 400 떨어진 위치에 카메라를 배치한다. </u>

<u>FollowCamera는 컨트롤러에 따라 CameraArm 회전을 하지 않으며 CameraBoom 아래 소켓에 부착되어 있다. </u>

<br> 

![이미지](/img/img1_2.JPG)

GetCapsuleComponent()->SetCapsuleSize(17.f, 64.f);와 같이 코드를 통해 캐릭터 캡슐 크기를 변경할 수 있다.

<br>

bUseControllerRotationYaw, Pitch, Roll은 카메라와 캐릭터가 동시에 회전하는 것을 막는다. 이 두개가 같이 돌아갈 경우 게임 중 캐릭터 앞 모습을 절대 볼 수 없다. 예를 들어 게임 중 앞으로 이동 중 뒤를 보는 것이 불가능하다.

<br>



2. 캐릭터 이동 및 회전

```c++
//키 입력 시 1초동안 65씩 회전함
	BaseTurnRate = 65.f;
	BaseLookUpRate = 65.f;

	GetCharacterMovement()->bOrientRotationToMovement = true; //bOrientRotationToMovement는 자동으로 캐릭터의 이동방향에 맞춰, 회전 보간을 해준다.
	GetCharacterMovement()->RotationRate = FRotator(0.0f, 540.f, 0.0f); 
	GetCharacterMovement()->JumpZVelocity = 650.f; //점프하는 힘
	GetCharacterMovement()->AirControl = 0.2f; //중력의 힘
}

// Called to bind functionality to input
void ANelia::SetupPlayerInputComponent(UInputComponent* PlayerInputComponent)
{
	Super::SetupPlayerInputComponent(PlayerInputComponent);					//키보드 입력 값을 전달받는 폰의 함수
	check(PlayerInputComponent);											//입력받은 키를 확인


	//입력 받은 키에 따라 해당 이름에 맞는 함수 호출? JUMP, 이동, 캐릭터 회전등 
	PlayerInputComponent->BindAction("Jump", IE_Pressed, this, &ANelia::Jump);				
	PlayerInputComponent->BindAction("Jump", IE_Released, this, &ACharacter::StopJumping);

	PlayerInputComponent->BindAction("Sprint", IE_Pressed, this, &ANelia::ShiftKeyDown);
	PlayerInputComponent->BindAction("Sprint", IE_Released, this, &ANelia::ShiftKeyUp);

	//PlayerInputComponent->BindAction("LMB", IE_Pressed, this, &ANelia::LMBDown);
	//PlayerInputComponent->BindAction("LMB", IE_Released, this, &ANelia::LMBUp);

	PlayerInputComponent->BindAxis("MoveForward", this, &ANelia::MoveForward);
	PlayerInputComponent->BindAxis("MoveRight", this, &ANelia::MoveRight);

	PlayerInputComponent->BindAxis("turn", this, &APawn::AddControllerYawInput);
	PlayerInputComponent->BindAxis("LookUp", this, &APawn::AddControllerPitchInput);
	PlayerInputComponent->BindAxis("TurnRate", this, &ANelia::TurnAtRate);
	PlayerInputComponent->BindAxis("LokkUpRate", this, &ANelia::LookUpAtRate);
}

void ANelia::MoveForward(float Value)
{
	//bMovingForward = false;?
	if ((Controller != nullptr)) //&& (Value != 0.0f) && (!bAttacking) && (MovementStatus != EMovementStatus::EMS_Death))
	{
		//find out which way is forward
		const FRotator Rotation = Controller->GetControlRotation();
		const FRotator YawRotation(0.f, Rotation.Yaw, 0.f);

		const FVector Direction = FRotationMatrix(YawRotation).GetUnitAxis(EAxis::X);
		AddMovementInput(Direction, Value);

		//bMovingForward = true;
	}
}

void ANelia::MoveRight(float Value)
{
	//bMovingRight = false;
	if ((Controller != nullptr) && (Value != 0.0f)) //&& (!bAttacking) && (MovementStatus != EMovementStatus::EMS_Death))
	{
		//find out which way is forward
		const FRotator Rotation = Controller->GetControlRotation();
		const FRotator YawRotation(0.f, Rotation.Yaw, 0.f);

		const FVector Direction = FRotationMatrix(YawRotation).GetUnitAxis(EAxis::Y);
		AddMovementInput(Direction, Value);

		//bMovingRight = true;
	}
}

//키를 누르고 있으면 컨트롤러가 1초안에 65도 회전 가능   ///getworld?
void ANelia::TurnAtRate(float Rate)
{
	AddControllerYawInput(Rate * BaseTurnRate * GetWorld()->GetDeltaSeconds());
}

void ANelia::LookUpAtRate(float Rate)
{
	AddControllerPitchInput(Rate * BaseLookUpRate * GetWorld()->GetDeltaSeconds());
}
```

GetCharacterMovement()->bOrientRotationToMovement = true;는 캐릭터 이동 방향에 맞춰 캐릭터를 회전 시켜주는 역할을 하며 점프 시 650만큼 위로 올라가고 중력0.2에 따라 아래로 떨어진다.

<br>

![이미지](/img/img1_3.JPG)

위처럼 각각 액션매핑은 점프와 스프린트 축매핑은 W,A,S,D와 마우스 이동에 따른 화면 회전 그리고 캐릭터 이동 키 입력에 따라 이동하는 방향을 캐릭터가 바라보도록 구현했다.

<br>

3. 캐릭터 애니메이션

![이미지](/img/img1_4.JPG)

블렌드 스페이스 제작

<iframe width="560" height="315" src="https://www.youtube.com/embed/dun4MAuNAOA" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

각각 스팟마다 idle, walk, run을 넣어주고 가운데 초록색 스팟을 이동시키면서 확인할 수 있다.

<br>

![이미지](/img/img1_5.JPG)

MainAnim_BP는 메인 캐릭터인 Nelia의 전반적인 움직임을 나타낸 블루프린트로 idle,walk,run,sprinting,jump 각각의 기능을 나타내었다.

<br>

![이미지](/img/img1_6.JPG)

첫 번째로 Idle/Walk/Run(스테이트)는 앞서 만든 블렌드 스페이스를 활용하여 구현하였고 Movement Speed가 증가할 때마다 블렌드 스페이스 내 속도가 증가하여 idle,walk,run 순서대로 진행하게 한다.

<br>

![이미지](/img/img1_7.JPG)

![이미지](/img/img1_8.JPG)

두 번째는 sprinting 다운 받은 애니메이션을 연결하여 구현하였고 blendspace와 sprinting 그리고 sprinting과 blendspace 사이 조건이 존재하는데 Nelia의 Movement Status가 Sprinting일 때 상태가 sprinting으로 변하고 sprinting이 아닌 경우 다시 idle/walk/run 상태의 블렌드 스페이슬로 돌아오게 됩니다. 코드를 보면 UENUM(열겨형)으로 작성하여 MovementStatus 상태를 Normal, Sprinting, Dead 3가지 중 고를 수 있는 것을 볼 수 있다.

<br>

![이미지](/img/img1_9.JPG)

![이미지](/img/img1_10.JPG)

세 번째는 Jump인데 다른 것은 위 설명과 비슷하고 다른 것이 있다면 Jump는  점프 이후 모션과 착지 이후 모션의 변화가 필요하기 때문에 위 사진처럼 착지하는 시간을 확인하여 위 사진처럼 0.5보다 클 때 착지한 것으로 보고 애니메이션을 다시 Idle/Walk/Run으로 진행한다.

<br>

__영상 볼 때 확인해야할 사항__

1. 카메라 회전 시 controller가 같이 회전이 되는 지 안 되는지

2. 캐릭터 방향 전환에 따라 이동하는 방향을 바라보고 있는지

3. 이동 애니메이션이 자연스러운지

4. 캐릭터가 장해물에 가려졌을 때 캐릭터가 줌인 되어 보여지는 지

   <iframe width="560" height="315" src="https://www.youtube.com/embed/2MdJDmu7BNU" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

