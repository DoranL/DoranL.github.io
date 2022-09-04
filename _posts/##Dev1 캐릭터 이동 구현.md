##Dev1 캐릭터 이동 구현

c++ 클래스에서 Critter이라는 Pawn 클래스를 생성  

<Critter.h>

```c++
    UPROPERTY(EditAnywhere)
	class UStaticMeshComponent* MeshComponent;
```

어디서든 편집 가능한 속성이며 StaticMesh 인스턴스를 생성하는데 사용한다.  

UPROPERTY 속성은 아래에서 더 알아보도록 하자.  

<Critter.cpp>

```c++
#include "Components/StaticMeshComponent.h"
    
    RootComponent = CreateDefaultSubobject<USceneComponent>(TEXT("RootComponent"));
	MeshComponent = CreateDefaultSubobject<UStaticMeshComponent>(TEXT("MeshComponent"));
	MeshComponent->SetupAttachment(GetRootComponent());
```

마지막 줄을 사용하기 위해서는 "Components/StaticMeshComponent.h"를 추가해야 한다.  

다음 해볼 것은 플레이어에게 카메라를 부착시키는 것을 해볼 것이다.  

<Critter.h>

```c++
    UPROPERTY(EditAnywhere)
	class UCameraComponent* Camera;
```

클래스 디폴트에서도 고칠 수 있고 월드에서도 고칠 수 있는 Camera 컴포넌트 생성  

<Critter.cpp>

```c++
#include "Camera/CameraComponent.h"
    Camera = CreateDefaultSubobject<UCameraComponent>(TEXT("Camera"));
	Camera->SetupAttachment(GetRootComponent());
	Camera->SetRelativeLocation(FVector(-300.f, 0.f, 300.f));
	Camera->SetRelativeRotation(FRotator(-45.f, 0.f, 0.f));
```

정확히는 cpp 작성 이후 블루프린트에 카메라 컴포넌트 생성 Critter 기준 x축 -300.f z축 300.f 위치에 카메라가 생성된다.

![이미지](img/DreamCatcher_camera_location.JPG)

