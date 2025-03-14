---
layout: single
title: "[Unity]캐릭터가 마우스 방향 바라보게 만들기"
categories: coding
tag: [Unity, C#]
toc: true
toc_sticky: true
toc_label: 목차
author_profile: false
excerpt: " "
---

# 마우스로 캐릭터 방향 바꾸기

탑다운 뷰 게임 중에는 캐릭터가 항상 마우스 방향을 바라보도록 구현된 경우가 있다.

{% include video id="2n_BinoS1Ug" provider="youtube" %}

유니티에서 게임 오브젝트를 원하는 방향으로 회전시키는 일은 벡터값만 구할 수 있다면 어렵지 않다.
하지만 마우스 방향을 바라보게 하는 경우, 사용자의 마우스가 움직이는 스크린 스페이스와
게임 월드 스페이스가 다른 공간이기 때문에 원하는 벡터값을 구하기 쉽지 않다.

## 월드 스페이스와 스크린 스페이스

![image_1]({{site.url}}/images/2025-03-14-lookmouse/ScreenWorldSpace.png)

유니티에는 "월드 스페이스"와 "스크린 스페이스" 두 가지 좌표 공간이 존재한다.

월드 스페이스는 실제 게임 세계가 만들어지는 좌표 공간이고, 모든 게임 오브젝트의 위치는 월드 스페이스
좌표 공간으로 확인 가능하다.

![image_3]({{site.url}}/images/2025-03-14-lookmouse/image_3.PNG)

스크린 스페이스는 유저가 바라보는 게임 화면의 좌표 공간으로, 보통 게임 UI를 구현하기 위해 사용한다.

![image_4]({{site.url}}/images/2025-03-14-lookmouse/image_4.PNG)

게임 화면의 마우스 위치를 알아내는 것은 어렵지 않지만, 이는 월드 스페이스가 아닌 스크린의 좌표값이기 때문에
플레이어 캐릭터가 이를 바라보게 하는 것은 불가능하다. 그렇다면 어떻게 마우스 위치를 파악하고 바라보게 할 수 있을까?

다양한 방법이 있겠지만 필자는 레이캐스팅(RayCasting)을 활용하여 구현하였다.

## 메인 카메라와 레이캐스팅

게임 스크린은 메인 카메라가 바라보는 방향의 화면을 담고 있다. 그리고 유니티에는 보이지 않는 레이저를 발사하여 충돌을 감지할 수 있는 레이캐스트라는 것이 존재한다.

이 둘을 조합하면 메인 카메라에서 마우스 위치를 기준으로 월드 스페이스에 레이캐스팅을 하여 충돌 좌표를 가져올 수 있다.

![image_2]({{site.url}}/images/2025-03-14-lookmouse/ScreenWorldSpace2.png)

코드로 구현하면 다음과 같다.

```c#
public void LookAtMouseCursor()
{
    //마우스 위치애서 카메라가 바라보는 방향으로 레이캐스팅
    Ray ray = mainCam.ScreenPointToRay(Input.mousePosition);
    RaycastHit hit; //충돌을 감지하는 요소
    if (Physics.Raycast(ray, out hit))
    {
        //레이 충돌 좌표
        Vector3 mouseDir = new Vector3(hit.point.x, transform.position.y, hit.point.z);
        transform.LookAt(mouseDir); //좌표 방향으로 회전
    }
}
```

## 구현 결과

![gif_1]({{site.url}}/images/2025-03-14-lookmouse/result.gif)
