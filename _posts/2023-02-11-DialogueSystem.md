---
layout: single
title: "DEV3 Dialogue System"
---

<span style = "color:green">Dialogue System편 내용 정리</span>

***

> E - Interact 키 
>
> 

***

<br/>

이번 장은 대화창 시스템 이해를 위해 작성하도록 하겠습니다. 

![이미지](/img/peliaback.JPG)

이번 장에서 다루는 기능들은 비교적 코드가 짧습니다.

<br/>

<mark>대화 시스템</mark>

1. 대화창 활성화 키(이후 대화는 마우스를 통해 선택 가능하도록 구현)

1. 위젯

1. 코드 설명


<br/>



<center>대화시스템 코드 설명</center>

대화시스템에서 사용되는 입력 키는 총 2가지 이며 E키와 마우스 왼쪽 클릭이다.

 E키 입력을 통해 대화창을 활성화 할 수 있고

![이미지](/img/7.JPG)

<br/>

각 NPC의 dialogue box에 플레이어가 검출되었을 때만 대화창이 출력될 수 있도록 레벨 블루프린트에서

CanUseDialogue bool변수를 통해 체크를 해주었습니다.

![이미지](/img/8.JPG)

<br/>

처음 시작되는 부분은 플레이어가 e키를 눌렀을 경우 <플레이어.cpp>의 Interact 함수로 이동

```c++
//e 눌렀을 때 첫 대화인 경우 여기에 있는 if문을 수행하게 된다.
		if (!MainPlayerController->bDialogueVisible)
		{
			MainPlayerController->DisplayDialogue();
		}
```

처음 e를 눌렀을 경우 맨 아래에 있는 if (!MainPlayerController->bDialogueVisible) 조건을 만족하고

<MainPlayerController.cpp>에 있는 DisplayDialogue()를 수행하게 됩니다.

<br/>

DisiplayDialogue() 함수는 카메라 고정 및 입력 키를 제한하고 대화창을 Visible로 설정해주는 함수로 

<UserInterface.cpp>에 있는 InitializeDialogue 함수에 매개변수로 IntroDialogue를 전달합니다.

```c++
void AMainPlayerController::DisplayDialogue()
{
	if (UserInterface)
	{
		bDialogueVisible = true;
		UserInterface->SetVisibility(ESlateVisibility::Visible);
		SetCinematicMode(true, true, true);

		//
		FInputModeGameAndUI InputModeUIOnly;

		SetInputMode(InputModeUIOnly);

		bShowMouseCursor = true;
		UserInterface->InitializeDialogue(IntroDialogue);
	}
}
```

<br/>

여기서 IntroDialogue는 데이터 테이블이며 행마다 캐릭터 이름과 메시지와 메시지에 대한 선택지를 둘 수 있는 구조로 구성되어 있습니다.

![이미지](/img/9.JPG)

<br/>

InitializeDialogue 함수는 변수들을 초기화하고 각 대사들을 넣어준다.

if문을 통해 대화가 있는지 확인하고 있을 경우 대화창을 화면에 보여주고 각 대사를 한 글자씩 출력하기 위해

0.5f초의 타이머를 설정한다.

```c++
void UUserInterface::InitializeDialogue(class UDataTable* DialogueTable)
{
	CurrentState = 0;

	CharacterNameText->SetText(FText::FromString(""));
	PlayerDialogTextBlock->SetText(FText::FromString(""));

	OnResetOptions();

	for (auto it : DialogueTable->GetRowMap())
	{
		FNPCDialogue* Row = (FNPCDialogue*)it.Value;

		Dialogue.Add(Row); // 새로운 대사 행들 추가
	}

	if (Dialogue.Num() > 0)
	{
		RowIndex = 0;

		CharacterNameText->SetText(FText::FromString(Dialogue[RowIndex]->CharacterName.ToString()));

		if (Dialogue[RowIndex]->Messages.Num() > 0)
		{
			MessageIndex = 0;

			OnAnimationShowMessageUI();

			GetWorld()->GetTimerManager().SetTimer(TimerHandle, this, &UUserInterface::OnTimerCompleted, 0.5f, false);
		}
	}
}
```

0.5초 마다 OnTimerCompleted 함수를 호출합니다.

<br/>

OnTimerCompleted 함수는 지정해둔 타이머를 지우고 AnimateMessage에 매개변수를 통해 대사 행에

문자열을 전달합니다.

<br/>

AnimateMessage 함수는 CurrentState를 1로 초기화 해주는데 CurrentState가 1이라는 것은 게임 내 대화창에 

위젯을 통해 지정해준 PlayerDialogueTextBlock에 입력받은 문자열을 한 글자씩 입력 중인 상태를 의미합니다.

해당 함수에서도 타이머를 사용해서 OnAnimationTimerCompleted를 수행합니다.

<br/>

OnAnimationTimerCompleted() 함수는 위에서 설명한 OnTimerCompleted 함수와 비슷하게 타이머를 중지하고 

입력 받은 문자열을 한 글자씩 OutputMessgage에 삽입하고 게임 내 대화창 위젯에 설정해준 PlayerDialogueTextBlock에 그 값을 한 글자씩 입력해줌 해당 줄이 있어야 위젯 상에 글자가 나타난다. 

출력해야할 글자가 남은 경우 다시 타이머를 켜고 OnAnimationTimerCompleted 함수를 사용하여 한 글자씩 화면에 출력해줍니다. 

해당 함수에서 모든 글자가 출력된 경우 CurrentState를 2로 초기화 해주는데 2의 의미는 해당 row 내에 있는 메시지의 해당된 대사가 모두 화면에 출력된 경우를 의미합니다.

<br/>

위에서 설명한 것 처럼 2인 상태로 다음 대화를 위해 마우스 왼쪽 클릭을 한 경우 

```c++
if (MainPlayerController->bDialogueVisible)
	{
		MainPlayerController->UserInterface->Interact();
	}
```

Interact 함수에 다시 들어가고 if(CurrentState == 2) 조건문을 수행합니다.

<br/>

위에서 대사가 남아있는지 확인하고 없는 경우 위젯에 나타나 있던 글자를 모두 초기화 이후 npc에 대한 대사 

선택지가 있는 경우 CharacterNameText를 플레이어 이름으로 설정 앞서 했던 것과 마찬가지로 해당 인덱스에

OptionText를 입력해줍니다. 이후 CurrentState ==3으로 지정 

<br/>

CurrentState ==3이란 npc 대사에 대한 플레이어가 선택할 수 있는 대사 선택지가 있는 경우 수행되는 

조건문으로 해당 조건문에서는 플레이어가 선택한 대사에 따라 다른 대화가 오고 갈 수 있도록 

사용자가 선택한 대사에 대한 AnswerIndex 값을 RowIndex에 넣어줌으로서 다음 대화를 달리 설정합니다.

<br/> 모든 대화가 종료되면 CurrentState를 다시 0으로 초기화 해주고 대화창을 숨기고 RemoveDialogue() 함수를 

호출하는데 해당 함수는 처음 대화 시작시 DialogueDisplay() 함수를 통해 제한해두었던 카메라 회전과 입력 키

제한을 해제하는 함수입니다.

```c++
void AMainPlayerController::RemoveDialogue()
{
	if (UserInterface)
	{
		FInputModeGameOnly InputModeGameOnly;

		SetInputMode(InputModeGameOnly);

		bShowMouseCursor = false;

		bDialogueVisible = false;
		SetCinematicMode(false, true, true);
	}
}
```

<br/>

마지막으로 플레이어가 npc의 콜리전에 닿을 경우 플레이어를 보고 있던 카메라를 npc 자신을 보도록 하는 

파트를 설명하고 최근에 추가한 카메라 줌인 기능을 설명을 하며 마무리 하겠습니다.

![이미지](/img/10.JPG)

해당 이미지는 npc 블루프린트 내 이벤트그래프에 생성한 것으로 사용자가 맵 내 배치되어 있는 캐릭터(npc)와 

대화를 할 수 있다는 것을 알려주기 위해 npc DialogueBox에 플레이어가 감지되면 머리 위에 

Press "E" to Chat 문구를 출력하도록 설정하였고 카메라를 npc가 소유할 수 있도록 구성하기 위해

SetViewTargetWithBlend 함수를 통해 카메라 타깃을 플레이어에서 new view target인 npc로 설정하여 

블루프린트에서 설정 카메라 위치로 2초 동안 이동할 수 있도록 구성하였습니다.

<br/>



<mark>카메라 줌인</mark>

<center>Camera 줌인 설명</center>

프로젝트 세팅에서 아래 사진처럼 마우스 휠 축을 통해 조절 할 수 있도록 코드와 연결해주고 

![이미지](/img/11.JPG)

```C++
	PlayerInputComponent->BindAxis("CameraZoom", this, &ANelia::CameraZoom);
```

<br/>

마우스 휠을 움직임으로서 입력 받는 value값이 0이 아니고 컨트롤러가 널이 아닌 경우 해당 if문을 수행하고

마우스 휠을 통해 기존 TargetArmLength에 Value*+-ZoomSteps(30)만큼의 거리만큼 줌인 줌아웃이 가능하도록 설정했고

최대 줌인 줌아웃 값은 최소 100.f 최대 800.f 거리까지 가능합니다.



