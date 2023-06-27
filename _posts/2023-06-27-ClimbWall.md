---
layout: single
title: "DEV4 캐릭터 벽타기"
---

<span style = "color:green">캐릭터 벽타기 내용 정리</span>

***

> W,A,S,D - 이동
>
> spacebar - 벽에서 떨어지기

***



***

게임 진행 중 플레이어가 순찰 범위에 들어오면 정찰병이 적들에게 침입을 알리는 부분이 있는데 여기서 

벽을 타는 기능이 필요했다.

<br/>

<mark>벽을 오르는 기능을 구현</mark>

1. 벽을 오르는 애니메이션
2. 플레이어가 오를 수 있는 벽이 있는 지 확인
3. 정상 도달 했는지 확인
4. 점프 시 벽에서 떨어져야함

<br/>

---

<1. 벽을 오르는 애니메이션>

![이미지](/img/climb_blend.png)

블렌드 스페이스를 통해 상하좌우 이동 시 애니메이션을 만들고 

<br/>

애니메이션 블루프린트에서 AnimGraph를 통해 Idle/Walk/Run 상태에서 벽을 오르는 중인지 확인하는 

bool 변수인 isClimb을 통해 ClimbWall을 할 수 있도록 구현을 하면 된다.

![이미지](/img/locomotion_clilmb.jpg)

</br>

</br>

<2. 플레이어가 오를 수 있는 벽이 있는 지 확인>

![이미지](/img/climb_line.png)

초록선은 앞에 벽이 있는 지 확인하는 선이고 빨간선은 정상에 도착 시 벽을 잡고 올라서는 애니메이션을 수행하기 

위해 더 이상 오를 벽이 없는지 있는지 확인하는 선이다.

</br>

아래 코드를 통해 Visibility 상태의 대상을 감지하여 있으면 true 없으면 false 값을 반환한다.

```c++
void ANelia::CanClimb()
{
	if (!isClimbUp)
	{
		FHitResult Hit;
		FVector Start = GetActorLocation();
		//TraceDistance는 시작할 때 40.f을 대입
		FVector End = Start + (GetActorForwardVector() * TraceDistance);
		FVector arriveEnd = (Start + (GetActorForwardVector())+FVector(0.f, 0.f, 135.f) + (GetActorForwardVector() * TraceDistance));

		FCollisionQueryParams TraceParams;
		bool bHit = GetWorld()->LineTraceSingleByChannel(Hit, Start, End, ECC_Visibility, TraceParams);
		bool bArriveHit = GetWorld()->LineTraceSingleByChannel(Hit, Start, arriveEnd, ECC_Visibility, TraceParams);
}
```

</br>

<3. 정상 도달 했는지 확인>

정상 도착하면 bool변수 isClimbup은 true 정상이면 벽을 타고 있는 상태가 아니기 때문에 isClimb은 false 

값으로 설정해준다. 동시에 정상을 오르는 애니메이션인 ClimbTop_Two를 수행해준다. 

</br>

3초 후 정상에 도착하면 땅을 밟고 돌아 다녀야하기 때문에 EMovementMode를 Move_Walking으로 설정 해주고

캐릭터 이동 방향으로 회전 가능하도록 true를 준다.

```c++
UAnimInstance* AnimInstance = GetMesh()->GetAnimInstance();

		if (isClimb && !onClimbLedge)
		{

			//UE_LOG(LogTemp, Warning, TEXT("isTopHere"));
			isClimbUp = true;
			isClimb = false;
			AnimInstance->Montage_Play(ClimbTop_Two, 1.2f);

			FTimerHandle WaitHandle;

			GetWorld()->GetTimerManager().SetTimer(WaitHandle, FTimerDelegate::CreateLambda([&]()
				{
					//UE_LOG(LogTemp, Warning, TEXT("6"));
					isClimbUp = false;
					GetCharacterMovement()->SetMovementMode(EMovementMode::MOVE_Walking);
					GetCharacterMovement()->bOrientRotationToMovement = true;
				}), 3.f, false);
		}
```

</br>

<4. 점프 시 벽에서 떨어져야함>

만약 플레이어가 벽을 오르는 도중인지 확인하고 맞다면 플레이어가 jump 키를 입력했을 경우 x,y축 기준으로 

-500.f 속도로 이동시키고 정상 도착 했을 때와 마찬가지로 모드를 Move_Walking과 회전 방향을 이동 방향과 동일

하도록 해준다.

```c++
FVector LaunchVelocity;
	bool bXYOverride = false;
	bool bZOverride = false;
	
	if (MainPlayerController) if (MainPlayerController->bPauseMenuVisible) return;

	if ((MovementStatus != EMovementStatus::EMS_Death))
	{
		bJump = true;
		
		Super::Jump();

		if (isClimb)
		{
			LaunchVelocity.X = GetActorForwardVector().X * -500.f;
			LaunchVelocity.Y = GetActorForwardVector().Y * -500.f;
			LaunchVelocity.Z = 0.f;

			LaunchCharacter(LaunchVelocity, bXYOverride, bZOverride);

			isClimb = false;

			GetCharacterMovement()->SetMovementMode(EMovementMode::MOVE_Walking);
			GetCharacterMovement()->bOrientRotationToMovement = true; 
```

</br>

벽을 타는 키는 이동 키와 동일하며 벽을 오르는 경우 x축으로 이동하던 direction값을 z축으로 변경해주는 거 외

크게 달라지는 것은 없습니다. 

![이미지](/img/yaw.jpg)

