---
layout: single
title: "[Unity] Analytics Funnel을 활용한 유저 이탈률 분석"
categories:
tag: [TIL, Unity, C#]
toc: true
toc_sticky: true
toc_label: 목차
author_profile: false
excerpt: ""
---

# 📕학습 개요

오늘은 `Unity Analytics` 시스템을 활성화하고, 그 중 `Funnel` 기능을 활용하여 게임 플레이 유저들의 이탈률을 분석하는 방법을 알아보았다. 분석한 내용을 바탕으로 유저 이탈 지점을 정확히 찾아내고 게임을 개선하는 방법을 쉽게 알아낼 수 있는 강력한 기능이다.

---

# 📖학습 내용

---

## 애널리틱스 활성화하기

유니티 프로젝트에서 애널리틱스 서비스를 활성화하려면 다음과 같은 단계를 거쳐야 한다. 생각보다 간단하다.

### 1단계: 패키지 설치

Unity 에디터 상단 메뉴에서 Window > Package Manager > Unity Registry > Analytics 검색 후 설치

![image_1]({{site.url}}/images/2025-07/install.PNG)

---

### 2단계: 프로젝트 연결 및 서비스 활성화

Edit > Project Settings로 이동한 뒤 Services 선택.

Unity 조직(Organization)을 선택하고, Create project ID 또는 Use an existing Unity project ID를 클릭해 현재 Unity 프로젝트를 클라우드 서비스에 연결.

![image_1]({{site.url}}/images/2025-07/services.PNG)

**Dashboard** 버튼을 클릭하여 유니티 프로젝트 대쉬보드로 이동 > 단축키 추가 > **Analytics** 추가

![image_1]({{site.url}}/images/2025-07/dashboard.PNG)

---

### 3단계: 코드 초기화

이제 게임이 시작될 때 애널리틱스 서비스를 초기화하는 코드를 작성줘야 한다.

```c#
public class AnalyticsManager : MonoBehaviour
{
    async void Start()
    {
        try
        {
            // Unity Gaming Services(UGS) 초기화
            await UnityServices.InitializeAsync();
        }
        catch (ServicesInitializationException e)
        {
            Debug.LogError(e.ToString());
        }
    }
}
```

이제 모든 준비가 끝났으니, 본격적으로 유저 데이터를 분석해보자.

---

## 퍼널 분석이란?

**퍼널(Funnel)** 은 '깔때기'를 의미한다. 유저가 게임에 들어와 우리가 원하는 최종 목표(예: 튜토리얼 완료)에 도달하기까지의 과정을 마치 넓은 입구에서 좁은 출구로 향하는 깔때기처럼 표현한 분석 모델이다.

이 분석의 핵심은 각 단계를 통과하는 유저가 얼마나 되는지 추적하여, 유저들이 어느 지점에서 가장 많이 빠져나가는지, 즉 이탈률(Drop-off Rate)이 가장 높은 구간을 시각적으로 찾아내는 것이다.

---

## 퍼널 분석 3단계 프로세스

퍼널 분석은 특별한 코드가 아닌, 정확한 설계와 분석 도구 활용의 조합으로 이루어진다.

### 1단계: 유저 여정 설계 및 이벤트 정의

가장 먼저 분석하고 싶은 유저의 핵심 여정을 정해야 한다.

다음은 유저의 초반 이탈율을 분석하기 위해, 초반 게임 플레이를 단계별로 세분화 한 것이다.

1. 첫 번째 튜토리얼 클리어

2. 두 번째 튜토리얼 클리어

3. 스테이지 1-4 클리어

4. 세 번째 튜토리얼 클리어

5. ...

각 단계는 나중에 추적할 커스텀 이벤트(Custom Event)의 기반이 된다.

### 2단계: 코드에 커스텀 이벤트 심기

이제 설계한 각 단계가 끝나는 시점에 맞춰 Unity 프로젝트 코드에 이벤트를 전송하는 코드를 추가한다. `Unity.Analytics` 에서 제공하는 `RecordEvent` 메서드를 사용한다.

```c#

    // 레벨 클리어 시 호출할 커스텀 이벤트

    public void SendStageClearEvent(int stageId, int timeTaken)
    {
        CustomEvent stageClearEvent = new CustomEvent("StageClear")
        {
            {"stage_id", stageId },
            {"time_to_clear", timeTaken }
        };

        // 커스텀 이벤트 전송
        AnalyticsService.Instance.RecordEvent(stageClearEvent);

        Debug.Log($"Analytics 이벤트 전송: levelClear (Level: {stageId}, Time: {timeTaken}s)");
    }

    public void SendTutorialCompleteEvent(int tutorialId, int timeTaken)
    {
        CustomEvent tutorialCompleteEvent = new CustomEvent("TutorialComplete")
        {
            { "tutorial_id", tutorialId }, { "time_to_clear", timeTaken }
        };

        AnalyticsService.Instance.RecordEvent(tutorialCompleteEvent);
    }
```

위와 같이 수집하고자 하는 데이터를 매개변수로 추가하여 넣어줄 수 있다. 위에서는 클리어한 스테이지 및 튜토리얼의 ID, 그리고 클리어 한 시간을 매개변수로 넣어주었다.

하지만 이대로 실행하면 `Analytics`의 이벤트 관리자에서 커스텀 이벤트와 커스텀 매게변수를 식별할 수 없으므로, 직접 추가해줘야 한다.

프로젝트의 대시보드에서 **Analytics** > 이벤트 관리자로 이동

![image_1]({{site.url}}/images/2025-07/eventManager.PNG)

Add New 버튼 클릭 > Custom Event > Add Custom Event 창 오픈

![image_1]({{site.url}}/images/2025-07/addCustomEventBlank.PNG)

코드에서 작성한 커스텀 이벤트의 이름과 똑같이 **Event Name** 작성. **Description**은 자유롭게 작성.

![image_1]({{site.url}}/images/2025-07/addCustomEventFilled.PNG)

**+ Assign Parameter** 버튼 클릭 > **+ Add new Parameter** > 추가한 매개변수 이름과 타입을 정확히 작성.

![image_1]({{site.url}}/images/2025-07/addCustomParameter.PNG)

모든 매개변수를 추가했다면, **Enable event** 옵션을 체크해주고 **Confirm** 버튼 클릭. 이제 이벤트 관리자에서 커스텀 이벤트를 받아들일 준비가 완료되었다.

### 3단계: 대시보드에서 퍼널 생성 및 분석

이제 게임을 플레이하면 이벤트 브라우저에 이벤트 스택이 쌓이는 것을 볼 수 있다.

![image_1]({{site.url}}/images/2025-07/eventBrowser.PNG)

이벤트 데이터가 쌓이기 시작헸으니, 이제 분석을 시작할 차례.

Analytics > 퍼널 메뉴로 이동 > **+ Add New Funnel** 클릭

![image_1]({{site.url}}/images/2025-07/addNewFunnel.PNG)

이제 1단계부터 설계했던 이벤트들을 순서대로 추가하면 된다. **Add Step** 버튼을 통해 단계를 추가하고, **Add Parameter** 를 통해 단계의 조건을 추가할 수 있다.

![image_1]({{site.url}}/images/2025-07/funnel.PNG)

퍼널을 저장하면 아래와 같이 각 단계별 전환율과 이탈률을 한눈에 볼 수 있는 그래프가 생성된다. 단, 업데이트 되는 속도가 상당히 느리다는 단점이 있다..

![image_1]({{site.url}}/images/2025-07/funnelTable.PNG)
