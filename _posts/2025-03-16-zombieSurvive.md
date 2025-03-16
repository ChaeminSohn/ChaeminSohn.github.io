---
layout: single
title: "[Unity]좀비 서바이벌"
categories: coding
tag: [Unity, C#]
toc: true
toc_sticky: true
toc_label: 목차
author_profile: false
excerpt: "유니티로 만든 쿼터뷰 좀비 생존 게임"
---

**해당 게임은 이제민 저자의 '레트로의 유니티 게임 프로그래밍 에센스' 를 기반으로 제작되었음을 밝힙니다.**{: .notice--info}

# 게임 소개 및 기능

유니티를 통해 간단한 좀비 슈팅 서바이벌 게임을 만들어보았다.
플레이어는 무한히 생성되는 좀비를 피해다니며 총을 쏴서 제거해야 한다.

게임의 기능은 다음과 같다.

1. 맵에 있는 모든 좀비를 처치할때마다 새로운 웨이브가 생성되며, 갈수록 생성되는 좀비의 수가 많아진다.
2. 플레이어가 좀비를 처치할 때마다 점수가 올라간다.
3. 플레이어는 좀비에 접촉할 때마다 체력이 닳는다.
4. 플레이어는 맵에 랜덤으로 생성되는 아이템을 통해 체력, 탄약 등을 보충할 수 있다.
5. 플레이어는 재장전 버튼을 통해 탄창을 채울 수 있다.

최대한 많은 좀비를 처치하여 높은 기록을 남기는 것이 게임의 주된 목표라고 볼 수 있다.

# 게임 구현

## 메인 오브젝트 생성

게임 모델링은 '레트로의 유니티 게임 프로그래밍 에센스'에서 제공하는 에셋들을 사용하였다.

맵
![image_1]({{site.url}}/images/2025-03-16-zombieSurvive/image_1.PNG)

플레이어<br/>
![image_2]({{site.url}}/images/2025-03-16-zombieSurvive/image_2.PNG)

좀비<br/>
![image_3]({{site.url}}/images/2025-03-16-zombieSurvive/image_3.PNG)

## 플레이어 움직임

본 게임에서는 플레이어의 움직임을 총 두 가지 방식으로 구현했다.

첫 번째는 '레트로의 유니티 게임 프로그래밍 에센스' 에서 사용하는 방식으로,
w s 키 또는 ↑ ↓ 방항키로 캐릭터를 앞 뒤로 움직이고, a d 키 또는 ← → 방향키로 캐릭터를 회전시킨다.
그리고 마우스를 좌클릭 할 시 총알이 플레이어가 바라보는 방향으로 나간다.

```c#
///PlayerInput.cs
public string moveAxisName = "Vertical"; // 앞뒤 움직임을 위한 입력축 이름
public string rotateAxisName = "Horizontal"; // 좌우 회전을 위한 입력축 이름
public string fireButtonName = "Fire1"; // 발사를 위한 입력 버튼 이름
public string reloadButtonName = "Reload"; // 재장전을 위한 입력 버튼 이름
// 값 할당은 내부에서만 가능
public float move { get; private set; } // 감지된 움직임 입력값
public float rotate { get; private set; } // 감지된 회전 입력값
public bool fire { get; private set; } // 감지된 발사 입력값
public bool reload { get; private set; } // 감지된 재장전 입력값
// 매프레임 사용자 입력을 감지
private void Update() {
    // 게임오버 상태에서는 사용자 입력을 감지하지 않는다
    if (GameManager.instance != null && GameManager.instance.isGameover)
    {
        move = 0;
        rotate = 0;
        fire = false;
        reload = false;
        return;
    }
    // move에 관한 입력 감지
    move = Input.GetAxis(moveAxisName);
    // rotate에 관한 입력 감지
    rotate = Input.GetAxis(rotateAxisName);
    // fire에 관한 입력 감지
    fire = Input.GetButton(fireButtonName);
    // reload에 관한 입력 감지
    reload = Input.GetButtonDown(reloadButtonName);
    }
```

```c#
//PlayerMovement.cs
// 입력값에 따라 캐릭터를 앞뒤로 움직임
private void Move()
{
    Vector3 moveDistance =
        playerInput.move * transform.forward * moveSpeed * Time.deltaTime;
    playerRigidbody.MovePosition(playerRigidbody.position + moveDistance);
}
// 입력값에 따라 캐릭터를 좌우로 회전
private void Rotate()
{
    float turn = playerInput.rotate * rotateSpeed * Time.deltaTime;
    playerRigidbody.rotation *= Quaternion.Euler(0, turn, 0);
}
```

위 코드에서는 PlayerInput.cs 를 통해 사용자의 입력값을 받고, PlayerMovement.cs를 통해 움직임을 처리한다.

책에서는 플레이어 움직임 구현 정도야 간단하게 구현하고 넘어가기 위해 이런 방식을 선택했겠지만, 개인적으로는 플레이어의 움직임을 이런식으로 구현하는 것은 마음에 들지 않았다. 조작감이 매우 불편했기 때문이다. 플레이어의 앞, 뒤가 게임월드 안에 존재하는 캐릭터의 좌표계를 기준으로 하기 때문에, 게임을 플레이하는 사용자가 인식하는 앞, 뒤 방향과 다르기 때문이다. 가령 플레이어가 게임 화면의 6시 방향을 바라보고 있다면, 사용자가 앞 방향키를 눌러도 플레이어는 우리가 인식하는 뒤 방향으로 움직이게 된다. 이는 직관적이지 않고, 인풋이 직관적이지 않으면 조작감이 불편하다는 소리다.

심지어 좀비에게 총알을 맞추기 위해 플레이어를 직접 회전시켜서 조준을 해야 한다니, 빠르게 움직이는 좀비들을 말이다. 이런 조작방식으로 게임을 출시한다면 장담컨데 아무도 이 게임을 플레이 하지 않을 것이다.

한번이라도 탑뷰 혹은 쿼터뷰 게임을 해본 사람은 알겠지만, 보통 플레이어 캐릭터는 바라보는 방향과 무관하게 wasd 키에 맞춰 우리가 바라보는 게임 화면의 방향으로 움직인다. 개인적으로 가장 직관적이고 적응하기 편한 방식이라고 생각한다. 또한, 빠르게 움직이고 점점 늘어나는 좀비들에게 재빠르게 대응할 수 있도록 플레이어가 항상 마우스의 위치를 바라보도록 구현하는 것이 좋아보인다.

먼저, 플레이어 캐릭터가 게임 화면 방향을 기준으로 움직이게 만들어보자.

```c#
//PlayerMovement.cs
private void Move()
 {
     //카메라가 바라보는 기준(게임 뷰 기준)으로 캐릭터 움직임
     Vector3 vec = mainCam.transform.localRotation * Vector3.forward;
     vec.y = 0; //y벡터값 삭제
     Vector3 directionForward = vec.normalized; //게임 스크린 북쪽 방향
     Vector3 directioRight = Quaternion.Euler(0, 90, 0) * directionForward; //게임 스크린 동쪽 방향
     //플레이어가 움직일 방향 계산식
     Vector3 moveDistance =
         ((playerInput.moveVertical * directionForward)
             + (playerInput.moveHorizontal * directioRight)).normalized
             * moveSpeed * Time.deltaTime;
    //움직임 적용
     playerRigidbody.MovePosition(playerRigidbody.position + moveDistance);
 }
```

먼저, 게임 월드 스페이스와 스크린 스페이스(게임 화면 스페이스)를 대응하기 위해 메인 카메라의 로테이션 값을 사용한다. 이렇게 할 수 있는 이유는 메인 카메라가 항상 플레이어를 바라보게 만들 것이기 때문이다.

![image_4]({{site.url}}/images/2025-03-16-zombieSurvive/image_4.png)

이 게임은 쿼터뷰 형식을 사용하기 때문에, 벡터의 y값을 제거해줘야 플레이어가 바닥을 뚫고 들어가거나 하늘로 승천하는 불상사를 방지할 수 있다.

그리고 vec 값을 정규화시켜서 정면 방향 벡터를 얻고, 이 값을 y축 기준으로 90도 회전시켜서 오른쪽 방향 벡터를 얻을 수 있다. 이제 사용자 인풋값과 방향 벡터를 통해 플레이어 캐릭터가 이동할 벡터를 계산하고, 리지드바디를 통해 움직임을 적용해주면 끝이다.

![gif_1]({{site.url}}/images/2025-03-16-zombieSurvive/move.gif)

플레이어가 마우스의 위치를 바라보는 로직은 다음 포스트에서 구현되었다.
[플레이어가 마우스 위치 바라보게 만들기](https://chaeminsohn.github.io/coding/lookmouse/){: .btn .btn--info}

## 플레이어 애니메이션

이 게임에서 플레이어가 필요한 애니메이션은 다음과 같다.

1. 앞, 뒤로 움직이는 애니메이션
2. 좌, 우로 움직이는 애니메이션
3. 총을 조준하는 애니메이션 (Idle)
4. 총을 재장전 하는 애니메이션
5. 사망 애니메이션

![gif_2]({{site.url}}/images/2025-03-16-zombieSurvive/walkForward.gif)
![gif_3]({{site.url}}/images/2025-03-16-zombieSurvive/walkRight.gif)
![gif_4]({{site.url}}/images/2025-03-16-zombieSurvive/idle.gif)
![gif_5]({{site.url}}/images/2025-03-16-zombieSurvive/reload.gif)
![gif_6]({{site.url}}/images/2025-03-16-zombieSurvive/death.gif)

휴머노이드 형 모델의 애니메이션은 [mixamo.com](https://www.mixamo.com/#/){: .btn .btn--info}에서 간단하게 구할 수 있다.

문제는 플레이어가 두가지 행동을 동시에 할때, 예를 들어 뛰면서 재장전을 하는 경우 새로운 애니메이션 클립이 필요하게 된다. 이런식으로 모든 경우의 수를 따지면 필요한 애니메이션 클립의 수가 너무 많아지고 애니메이션 컨트롤이 복잡해지게 된다. 이럴땐 애니메이션 레이어를 나눔으로써 다양한 경우의 수에 대응할 수 있다.

### 애니메이션 레이어

게임 캐릭터 모델의 애니메이션 레이어를 나눠주면, 각각 다른 부위에 원하는 애니메이션 클립을 동시에 적용할 수 있다. 플레이어가 앞으로 움직이면서 총을 재장전 하는 애니메이션 클립을 따로 만들지 않아도 된다는 소리다.
일단 레이어를 구분하기 위해 아바타 마스크를 만들어줘야 한다.

![image_6]({{site.url}}/images/2025-03-16-zombieSurvive/image_6.PNG)

애니메이션을 적용할 부위만 초록색으로 표시되는 것을 볼 수 있다.
상체만 따로 아바타 마스크를 만들어서, 상체와 하체가 따로 애니메이션을 재생하도록 만들어보자.

![image_7]({{site.url}}/images/2025-03-16-zombieSurvive/image_7.PNG)

플레이어의 애니메이션 컨트롤러의 Layers 탭에 Upper Body 레이어를 추가해주고, 마스크를 설정해준다.
IK Pass 라는 것이 활성화 된 것을 볼 수 있는데, 이에 대해서는 뒤에서 설명하겠다.
이제 상체를 움직이는 애니메이션과 하체를 움직이는 애니메이션을 병합해서 동시에 재생할 수 있다.

### 블렌드 트리

블렌드 트리는 여러 애니메이션 클립을 파라미터값에 따라 적절히 혼합하여 사용한다. 캐릭터의 이동 모션을 블렌드 트리를 사용하여 구현하면 조금 더 자연스러운 움직임을 구현할 수 있다.

![image_8]({{site.url}}/images/2025-03-16-zombieSurvive/image_8.PNG)
![image_9]({{site.url}}/images/2025-03-16-zombieSurvive/image_9.PNG)

앞,뒤 방향과 좌,우 방향 입력 방식이 다르므로 파라미터를 두 가지 사용하는 2D Directional의 블렌드 타입을 사용했다. 게임 플레이어의 wasd 인풋에 따라 moveH, moveV 값이 -1 에서 1 사이의 값을 가지게 되고, 이에 맞춰서 앞,뒤,좌,우 로 움직이는 애니메이션이 적절히 섞이게 된다.
예를 들어, 앞 방향키와 오른쪽 방향키를 누르면 플레이어가 앞으로 가는 애니메이션과 오른쪽으로 가는 애니메이션이 반반씩 섞여서 재생된다.

### 총의 위치 고정

플레이어가 어떤 애니메이션을 사용하던 상관없이 캐릭터의 손의 위치와 총의 손잡이의 위치가 항상 동기화되도록 만들기 위해, 애니메이터의 IK를 사용했다.

IK(Inverse Kinematics)는 FK(Forward Kinematics)의 반대 개념으로 볼 수 있다.
캐릭터 애니메이션은 기본적으로 FK 방식으로 동작하며, 이는 부모 조인트에서 자식 조인트 순서대로 움직임이 적용된다는 뜻이다. 만약 손가락으로 무언가를 가르키는 애니메이션을 재생한다면, 어깨, 팔, 손, 손가락 순서대로 움직임이 적용된다. 반대로 IK는 자식 조인트의 위치를 먼저 결정하고 부모 조인트가 거기에 맞춰 변형된다.
IK를 사용하려면 먼저 애니메이터 컨트롤러의 레이어에서 IK Pass 설정이 켜져 있어야 한다.

```c#
// 애니메이터의 IK 갱신
private void OnAnimatorIK(int layerIndex)
{
    //총의 기준점을 3D모델의 오른쪽 팔꿈치 위치로 이동
    gunPivot.position = playerAnimator.GetIKHintPosition(AvatarIKHint.RightElbow);
    //왼손의 위치와 회전을 총의 왼쪽 손잡이에 맞춤
    playerAnimator.SetIKPositionWeight(AvatarIKGoal.LeftHand, 1.0f);
    playerAnimator.SetIKRotationWeight(AvatarIKGoal.LeftHand, 1.0f);
    playerAnimator.SetIKPosition(AvatarIKGoal.LeftHand, leftHandMount.position);
    playerAnimator.SetIKRotation(AvatarIKGoal.LeftHand, leftHandMount.rotation);
    //오른손의 위치와 회전을 총의 오른쪽 손잡이에 맞춤
    playerAnimator.SetIKPositionWeight(AvatarIKGoal.RightHand, 1.0f);
    playerAnimator.SetIKRotationWeight(AvatarIKGoal.RightHand, 1.0f);
    playerAnimator.SetIKPosition(AvatarIKGoal.RightHand, rightHandMount.position);
    playerAnimator.SetIKRotation(AvatarIKGoal.RightHand, rightHandMount.rotation);
}
```

OnAnimatorIK() 메서드는 IK 정보가 갱신될 때마다 자동으로 실행된다.

![gif_7]({{site.url}}/images/2025-03-16-zombieSurvive/gunFollow.gif)

플레이어의 양손에 총이 잘 위치된 것을 볼 수 있다.

## 좀비 구현

좀비는 항상 플레이어의 위치를 알아내어 추적해야 한다. 무작정 플레이어 방향으로 일직선상 움직이지 않고, 최적의 경로를 계산하기 위해 유니티 내비게이션 시스템을 사용해보자.

또한, 똑같은 좀비만 생성되는 것은 재미없으므로 스크립터블 오브젝트를 통해 다양한 타입의 좀비를 만들어보자.

### 유니티 내비게이션 시스템

유니티는 다른 위치로의 최적 경로를 계산하고 장애물을 피하며 이동하는 인공지능을 생성하는 내비게이션 시스템을 제공한다. 이를 사용하면 타겟(플레이어)를 추적하는 좀비 AI를 쉽게 구현할 수 있다.

일단 내비게이션을 사용하려면 내비메시(NavMesh)를 미리 구워줘야 한다. 원하는 게임 필드에 NavMesh Surface 컴포넌트를 추가하고, Agent Settings를 원하는 대로 바꿔준 뒤 Bake 버튼을 눌러주면 된다. 여기서 Agent Settings를 통해 게임 필드를 돌아다닐 에이전트의 타입, 키와 반지름, 올라갈 수 있는 계단 높이 등을 설정할 수 있는데, 이를 통해 에이전트가 이동할 수 있는 영역을 자동으로 측정할 수 있다.

![image_10]({{site.url}}/images/2025-03-16-zombieSurvive/image_10.PNG)

이제 스크립트를 통해 네비게이션 에이전트의 추적 대상만 할당해주면 끝이다.

```c#
//Zombie.cs
// 살아 있는 동안 무한 루프
while (!dead)
{
    if (hasTarget)
    {
        //추적 대상이 존재하는 경우 : 내비게이션 활성화
        navMeshAgent.isStopped = false;
        navMeshAgent.SetDestination(targetEntity.transform.position);
    }
```

### 좀비 데이터

우선 좀비의 데이터를 스크립터블 오브젝트로 구현해보자.

```c#
[CreateAssetMenu(menuName = "Scriptable/ZombieData", fileName = "Zombie Data")]
public class ZombieData : ScriptableObject {
    public float health = 100f; // 체력
    public float damage = 20f; // 공격력
    public float speed = 2f; // 이동 속도
    public Color skinColor = Color.white; // 피부색
}
```

이를 스크립터블 오브젝트로 구현하는 이유는 다음과 같다.

1. 여러 오브젝트가 공유하여 사용할 데이터이다.
2. 데이터를 유니티 인스펙터 창에서 편집 가능하다.

이런 식으로 어떤 오브젝트의 데이터를 스크립터블 오브젝트로 분리하면 관리하기도 편하고, 다수의 오브젝트가 하나의 데이터 오브젝트만 참고하므로 메모리 사용량에서도 효과적이다.

![image_11]({{site.url}}/images/2025-03-16-zombieSurvive/image_11.PNG)
![image_12]({{site.url}}/images/2025-03-16-zombieSurvive/image_12.PNG)
![image_13]({{site.url}}/images/2025-03-16-zombieSurvive/image_13.PNG)

이런 식으로 같은 ZombieData 타입을 여러개 만들고, 속성값을 다르게 할당하여 다양한 종류의 좀비를 만들 수 있다.

```c#
//Zombie.cs
// 좀비의 초기 스펙을 결정하는 셋업 메서드
public void Setup(ZombieData zombieData)
{
    //좀비 데이터를 기반으로 좀비의 능력치 설정
    startingHealth = zombieData.health;
    health = zombieData.health;
    damage = zombieData.damage;
    navMeshAgent.speed = zombieData.speed;
    zombieRenderer.material.color = zombieData.skinColor;
}
```

### 좀비 생성기

이제 좀비를 맵에 계속 생성해주는 좀비 생성기를 만들어 보자. 좀비 생성기가 수행할 임무는 다음과 같다.

1. 좀비 전부 사망 시 새로운 웨이브 생성
2. 웨이브 횟수 카운팅, 점점 많은 좀비 웨이브 생성
3. 좀비의 데이터, 스폰 위치를 각자 랜덤으로 지정
4. 현재 살아 있는 좀비들을 리스트를 통해 관리

```c#
public class ZombieSpawner : MonoBehaviour
{
    public Zombie zombiePrefab; // 생성할 좀비 원본 프리팹

    public ZombieData[] zombieDatas; // 사용할 좀비 셋업 데이터들
    public Transform[] spawnPoints; // 좀비 AI를 소환할 위치들

    private List<Zombie> zombies = new List<Zombie>(); // 생성된 좀비들을 담는 리스트
    private int wave; // 현재 웨이브
```

좀비들을 리스트에 담는 이유는 좀비들의 총 수가 웨이브가 지나갈 수록 점점 커지기 때문이다.

```c#
 private void Update()
    {
        // 게임 오버 상태일때는 생성하지 않음
        if (GameManager.instance != null && GameManager.instance.isGameover)
        {
            return;
        }

        // 좀비를 모두 물리친 경우 다음 스폰 실행
        if (zombies.Count <= 0)
        {
            SpawnWave();
        }
    }
}
   // 현재 웨이브에 맞춰 좀비들을 생성
    private void SpawnWave()
    {
        wave++;
        //현재 웨이브 * 1.5를 반올림한 값만큼 좀비 생성
        int spawnCount = Mathf.RoundToInt(wave * 1.5f);

        for (int i = 0; i < spawnCount; i++)
        {
            CreateZombie();
        }
    }

    // 좀비를 생성하고 생성한 좀비에게 추적할 대상을 할당
    private void CreateZombie()
    {
        //좀비의 데이터, 생성 위치 랜덤으로 지정
        ZombieData zombieData = zombieDatas[Random.Range(0, zombieDatas.Length)];
        Transform spawnPoint = spawnPoints[Random.Range(0, spawnPoints.Length)];
        //좀비 프리팹으로부터 좀비 생성
        Zombie zombie = Instantiate(zombiePrefab, spawnPoint.position, spawnPoint.rotation);

        zombie.Setup(zombieData);

        zombies.Add(zombie);
    }
```

좀비의 스폰 위치를 맵 전체로 지정하면 간혹 플레이어 위치에 좀비가 생성되는 불상사가 발생할 수 있으므로 4개의 스폰 지점을 미리 지정해주었다.

![gif_8]({{site.url}}/images/2025-03-16-zombieSurvive/zombies.gif)

다른 타입의 좀비가 생성되어 플레이어를 추적하는 모습을 볼 수 있다.

## 총 발사 처리

총 발사 및 피격 처리는 레이케스팅을 통해 구현하였다.<br/>
실제로 총알 오브젝트를 발사하여 피격 오브젝트 입장의 OnCollision 메서드를 통해 데미지를 입히는 방식도 사용할 수 있지만, 레이케스팅을 활용하는 방식이 훨씬 가볍고 간단하다.

```c#
//Gun.cs
// 실제 발사 처리
 private void Shot()
 {
     RaycastHit hit;
     Vector3 hitPosition = Vector3.zero;
     if (Physics.Raycast(fireTransform.position, fireTransform.forward, out hit, fireDistance))
     {
         IDamageable target = hit.collider.GetComponent<IDamageable>();
         if (target != null)
         {
             target.OnDamage(gunData.damage, hit.point, hit.normal);
         }
         hitPosition = hit.point;
     }
     else
     {
         //레이케스트 미충돌 시
         //탄알이 최대 사정거리까지 나아간 지점을 충돌 위치로 사용
         hitPosition = fireTransform.position + fireTransform.forward * fireDistance;
     }
     StartCoroutine(ShotEffect(hitPosition));
     if (--magAmmo <= 0)
     {
         state = State.Empty; //탄창에 남은 탄알이 없음
     }
 }
 // 발사 이펙트와 소리를 재생하고 탄알 궤적을 그림
 private IEnumerator ShotEffect(Vector3 hitPosition)
 {
     muzzleFlashEffect.Play();
     shellEjectEffect.Play();
     gunAudioPlayer.PlayOneShot(gunData.shotClip);
     bulletLineRenderer.SetPosition(0, fireTransform.position);
     bulletLineRenderer.SetPosition(1, hitPosition);
     // 라인 렌더러를 활성화하여 탄알 궤적을 그림
     bulletLineRenderer.enabled = true;
     // 0.03초 동안 잠시 처리를 대기
     yield return new WaitForSeconds(0.03f);
     // 라인 렌더러를 비활성화하여 탄알 궤적을 지움
     bulletLineRenderer.enabled = false;
 }
```

IDamageable 오브젝트 타입은 이 게임에서 데미지를 입을 수 있는 모든 객체들이 상속받는 인터페이스이다.

```c#
public interface IDamageable
{
    void OnDamage(float damage, Vector3 hitPoint, Vector3 hitNormal);
}
```

# 실행 결과

{% include video id="YHCbe7MXQBw" provider="youtube" %}
