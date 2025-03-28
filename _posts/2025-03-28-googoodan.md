---
layout: single
title: "[C#]구구단 출력"
categories: coding
tag: [C#]
toc: true
toc_sticky: true
toc_label: 목차
author_profile: false
excerpt: " "
---

# 구현 목표

이중 반복문을 사용하여 2단부터 9단까지의 구구단을 출력하는 프로그램을 작성.

## 세로로 출력

```c#
for(int i = 1; i <= 9; i++)
{
    for(int j = 2; j <= 9; j++)
    {
        Console.Write(j + " x " + i + " = " + i * j + "\t");
    }
    Console.WriteLine();
}
```

## 결과

![image1]({{site.url}}/images/2025-03-28-googoodan/googoodan_1.PNG)

## 가로로 출력

```c#
for(int i = 2; i <= 9; i++)
{
    for(int j = 1; j <= 9; j++)
    {
        Console.Write(i + " x " + j + " = " + i * j + "\t");
    }
    Console.WriteLine();
}
```

## 결과

![image2]({{site.url}}/images/2025-03-28-googoodan/googoodan_2.PNG)
