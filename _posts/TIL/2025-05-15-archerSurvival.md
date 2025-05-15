---
layout: single
title: "[Unity]미니 프로젝트 - 아처 서바이벌"
categories: coding
tag: [Unity, C#]
toc: true
toc_sticky: true
toc_label: 목차
author_profile: false
excerpt: ""
---

---

## 🎯게임 소개 및 기능

- **장르** : 궁수의 전설 + 뱀파이어 서바이벌의 아이디어를 합친 모방 게임
- **개발 기간** : 2025.05.08 ~ 2025.05.14(약 5일)
- **개발 인원** : 5인
- **사용 툴** : Unity
- **주요 기능** :
  - 플레이어 조작(이동, 체력, 스탯 관리)
  - 몬스터와 전투(AI, FSM)
  - 스킬 시스템(투사체, 폭발)
  - 스테이지 시스템(몬스터 스폰, 무한 모드)
  - 업적 시스템
  - NPC 대화 등

---

## ⚙️주요 기능 구현 & 문제 해결

### 1. 플레이어 이동 기능과 대쉬 기능의 충돌

**기존 접근**

- Player Move는 이동에 대한 입력을 받고 이동 방향과 속도를 곱해서 Rigidbody의 Velocity를 통해 구현됨
- Player Dodge 기능은 키 입력을 받아서 일정 시간 동안 Dodge 속도를 곱해서 Velocity를 강제로 수정하여 회피하도록 구현됨

**문제**

- Player Dodge 기능을 사용할 경우 Player Move의 Velocity와 충돌하는 문제가 발생하여 플레이어가 계속해서 속도를 받거나 잠시 멈춤 현상이 발생.

**과정**

- Dodge 이후에 다시 현재 이동 속도를 반영해줬다.
  문제점 : DodgeSpeed와 Duration을 이용한 Dodge 값 커스텀 자체가 복잡해지며, Animation의 속도를 맞춰주기 어렵다.

**해결**

- RigidBody의 MovePosition을 이용하면 정확한 포지션 제어가 가능함과 동시에 충돌이 적용되기 때문에 Dodge 기능에 적합하다.
- 여기에 Lerp를 사용하여 MovePosition을 해주면 애니메이션과 물리 위치가 자연스럽게 맞아 떨어지게 된다.

```c#
IEnumerator DodgeRoutine(Vector2 direction, float dodgePower, float duration, float coolTime)
{
    isDodging = true;
    isDodgeCoolDown = true;
    float elapsed = 0f;
    Vector2 start = pRigidbody.position;
    Vector2 end = start + direction.normalized * dodgePower;
    while (elapsed < duration)
    {
        elapsed += Time.fixedDeltaTime;
        float t = elapsed / duration;
        Vector2 newPosition = Vector2.Lerp(start, end, t);
        pRigidbody.MovePosition(newPosition);
        yield return new WaitForFixedUpdate();
    }
    isDodging = false;
    playerStat.isInvincible = false;
    yield return new WaitForSeconds(coolTime);
    isDodgeCoolDown = false;
}
```

---

### 2. 싱글톤 객체의 중복된 이벤트 구독

**발생 문제**

- 싱글톤 오브젝트의 Awake 에서 Destroy 가 제때 이루어지지 않아 Start 에서 불필요한 이벤트 구독이 다수 발생하는 현상

**원인**

- Destroy 메서드는 해당 게임 오브젝트를 "파괴 예정 목록"에 등록만 할 뿐, 파괴 처리는 나중에 일괄적으로 처리됨

**해결**

DestroyImmediate : 파괴 처리를 즉각적으로 처리하나, 런타임에 실행 시 버그 발생 가능성이 매우 큼. 따라사 사용하지 않는 것이 권장됨.
파괴 가능성이 있는 오브젝트에서 이벤트 구독을 할 시, OnDestroy 에서 반드시 구독 해제를 해줄 것

```c#
    private void Start()
    {
        playerResource = PlayerController.Instance.GetComponent<PlayerResource>();
        if(playerResource != null )
        {
            playerResource.OnGoldChanged += UpdateUI;
            UpdateUI();
        }
    }

    private void OnDestroy()
    {
        playerResource.OnGoldChanged -= UpdateUI;
    }
```

```c#
//card.cs
public void DisableCardInvoke()
{
    gameObject.SetActive(false);
    transform.rotation = Quaternion.Euler(0f, 0f, 0f); /카드 회전 초기화
    front.SetActive(false);
    back.SetActive(true);
}
```

---

### 3. GC(가비지 컬랙션) 발생 우려

**발생 문제**

- 상태 전환 시 new로 인한 GC(가비지 컬랙션) 발생 우려
- 몬스터마다 고유한 상태가 존재하며, 각 상태에 대한 행동을 FSM으로 구현하고 있을 때, 새로운 몬스터를 추가할 때마다 상속 구조가 점점 더 깊어져 복잡도가 커지는 문제가 발생

**원인**

- 상태 전환 시 Switch(new EnemyIdleState(stateMachine))처럼 매번 인스턴스 생성
- 새로운 몬스터 추가 시, 기존과 유사한 상태를 다시 구현해야 하는 비효율 발생
- 상속 구조가 깊어질수록 디버깅이 불편해짐

**해결**

- 상태들을 담는 딕셔너리 변수에 저장 후 상태들을 재사용
- BaseState에 중복 코드 구현
- FSM + Strategy 결합 패턴 고려

```c#
//EnemyStateMachine.cs
protected virtual void Awake()
 {
     States.Add(EENEMYSTATE.IDLE, new EnemyIdleState(this));
     States.Add(EENEMYSTATE.ATTACK, new EnemyAttackState(this));
     States.Add(EENEMYSTATE.DEAD, new EnemyDeadState(this));
     States.Add(EENEMYSTATE.STUN, new EnemyStunState(this));
 }
```

---

### 4. 델리게이트 이벤트 구독, 클로저

**발생 문제**

- for 문을 돌리는중 델리게이트 이벤트 구독시 매개변수를 넣어주면 최댓값이 들어가는 현상 발생

```c#
for(int i = 0; i < 3; i++)
{
	aButton[i].onClick.AddListener(() => mesod(i));
}
```

- 위 예제와 같이 i값을 넣으면 매개변수에 항상 2가 들어가는 현상

**원인** :

- 클로저로 인해 i는 하나의 공유된 변수로 처리
- C#에서는 람다식에서 사용된 외부변수는 캡쳐(Closure)됨
- 즉 i를 참조하는 람다들은 모두 하나의 i 변수만을 참조
  => 루프가 끝난 시점의 i값이 모든 람다에 적용

**해결**

- 참조가 필요할때마다 새로운 변수에 값을 할당해서 사용

```c#
for(int i = 0; i < 3; i++)
{
	int temp = i
	aButton[temp].onClick.AddListener(() => mesod(temp));
}
```

---

# ✅결과 및 마무리

## 🎮실행 결과 및 플레이 링크

{% include video id="V-Zsx7P0w1E" provider="youtube" %}

[게임 플레이](https://play.unity.com/en/games/14b48fe5-e079-45dd-a173-86e60d9d3d27/archer-survival)

## 🏁마무리하며

열정이 넘치는 팀원들을 만나서 1주일동안 매우 재미있게 작업할 수 있었고, 기능 구현도 멋지게 잘 된 것 같다. 하지만 코드 코드가 전체적으로 체계적이지 않아서 프로젝트를 진행할 수록 비교적 사소한 문제 해결에 지나치게 많은 시간이 쓰인다는 생각이 들었다. 이번 프로젝트를 통해 협업에서 코드 설계 부분의 중요성을 크게 느낀 것 같다.
