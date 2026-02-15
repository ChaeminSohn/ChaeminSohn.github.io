---
layout: single
title: "코루틴, 제대로 알고 사용하자"
categories:
tag: [Unity, C#]
toc: true
toc_sticky: true
toc_label: 목차
author_profile: false
excerpt: ""
---

# 📕학습 개요

유니티 개발 시 빈번하게 사용하는 `Coroutine` 을 단순히 "시간을 지연시킬 때 사용하는 함수" 정도로만 이해하고 사용하였는데, 이를 넘어 **.NET 에서 제공하는 IEnumerator 타입과 유니티 라이프 사이클이 어떻게 결합되어 동작하는지** 근본적인 원리를 파악해보고자 했다. 또한 멀티 쓰레딩과의 차이점을 명확히 하여 실무에서의 올바른 기준과 사용 패턴을 정리해보고자 한다.

---

# 📖학습 내용

---

## 2. 핵심 개념: C#의 Iterator와 yield

### 2.1. IEnumerator의 본질

- **정의:** `System.Collections`에 정의된 인터페이스로, 본래는 컬렉션(List, Array)을 순차적으로 순회하기 위해 만들어졌다.
- **구성 요소:** `Current`(현재 값), `MoveNext()`(다음 이동), `Reset()`(초기화).
- **foreach의 진실:** 우리가 사용하는 `foreach` 문은 컴파일러가 내부적으로 `IEnumerator`를 사용하는 `while` 문으로 변환(Lowering)하여 처리하는 '문법적 설탕(Syntactic Sugar)'이다.

### 2.2. yield 키워드의 역할 (C# Feature)

- **기능:** 함수의 실행을 일시 정지(Suspend)하고 호출자에게 제어권과 값을 반환(Yield)한다.
- **컴파일러의 마법 (State Machine):** `yield`가 포함된 함수는 컴파일 시 **별도의 클래스(상태 머신)**로 변환된다. 이 클래스는 지역 변수와 실행 위치(State)를 필드에 저장하여, 다음 호출 시 멈췄던 곳부터 정확히 이어서 실행할 수 있게 해준다.

---

## 3. 유니티는 이것을 어떻게 '시간 제어'로 바꿨나?

유니티는 C#의 **"데이터를 하나씩 꺼내고 멈추는"** 특성을 **"프레임 단위로 실행하고 멈추는"** 로직으로 응용했다.

### 3.1. 동작 메커니즘

1.  **StartCoroutine:** 유니티 엔진에게 `IEnumerator` 객체(상태 머신)를 등록한다.
2.  **Baton Pass:** 코루틴이 `yield return`을 만나면 실행을 멈추고 유니티 메인 루프에 제어권을 넘긴다.
3.  **Resume:** 유니티 엔진은 매 프레임 `YieldInstruction`(예: `WaitForSeconds`) 조건을 체크하고, 충족되면 `MoveNext()`를 호출하여 코루틴을 재개한다.

| 구분          | 일반 C# (Iterator)           | Unity (Coroutine)                   |
| :------------ | :--------------------------- | :---------------------------------- |
| **목적**      | 데이터 컬렉션 순회           | 게임 로직의 시간적 분할 실행        |
| **제어 주체** | 개발자 코드 (`foreach`)      | 유니티 엔진 (`PlayerLoop`)          |
| **반환값**    | 데이터 요소 (int, string 등) | 실행 재개 조건 (`YieldInstruction`) |

---

## 4. 심화: 코루틴 vs Async/Await & 쓰레드

### 4.1. 메인 쓰레드와 일관성 (Consistency)

- **코루틴은 싱글 쓰레드다:** 코루틴은 멀티 쓰레드처럼 보이지만, **메인 쓰레드**에서 유니티 라이프 사이클에 맞춰 실행된다.
- **이유:** 게임 로직의 데이터 무결성과 결정론적 결과(Determinism)를 보장하기 위해, 그리고 락(Lock) 비용으로 인한 성능 저하를 막기 위해 유니티는 메인 쓰레드 위주의 설계를 지향한다.

### 4.2. Async/Await와의 비교

| 특징          | Coroutine                    | Async / Await                       |
| :------------ | :--------------------------- | :---------------------------------- |
| **실행 위치** | Main Thread (필수)           | Worker Thread 가능 (Multi-thread)   |
| **주 용도**   | **시각적 연출, 프레임 제어** | **데이터 처리, 파일 I/O, 네트워크** |
| **반환값**    | 없음 (void와 유사)           | `Task<T>` (결과값 반환 가능)        |
| **예외 처리** | `try-catch` 적용 까다로움    | `try-catch` 완벽 지원               |

> **결론:** 오브젝트를 움직이거나 연출하는 것은 `Coroutine`, 무거운 연산이나 DB 통신은 `Async`를 사용한다.

---

## 5. 최적화 및 주의사항 (Best Practices)

### 5.1. 가비지 컬렉션(GC) 방지

`yield return new WaitForSeconds(1f);`를 루프 안에서 사용하면 매번 새로운 객체를 힙 메모리에 할당(Allocation)하게 되어 GC 스파이크를 유발할 수 있다.

```c#
// [Bad] 매번 new 할당
yield return new WaitForSeconds(0.1f);

// [Good] 캐싱하여 재사용
var wait = new WaitForSeconds(0.1f);
yield return wait;

### 5.2 생명 주기 관리

- 코루틴은 실행된 `GameObject` 가 비활성화되거나 파괴되면 함께 정지/소멸된다.
- 하지만 `MonoBebaviour` 가 살아잇는 한, 코루틴 객체는 유니티 내부 참조 체인(PlayerLoop -> DelayedCallManager) 에 의해 GC 되지 않고 유지된다.

## 6. 요약

- 코루틴은 마법이 아니다: C#의 IEnumerator 상태 머신을 유니티가 게임 루프에 맞춰 영리하게 활용한 패턴이다.

- 협력적 멀티태스킹: 쓰레드를 쪼개는 것이 아니라, 함수가 스스로 실행 권한을 '양보(Yield)'하는 방식이다.

- 적재적소: 화면을 그려야 한다면 코루틴, 데이터를 처리해야 한다면 Async를 선택하자.
```
