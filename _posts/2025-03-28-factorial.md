---
layout: single
title: "[C#]팩토리얼 계산"
categories: coding
tag: [C#]
toc: true
toc_sticky: true
toc_label: 목차
author_profile: false
excerpt: " "
---

# 구현 목표

사용자로부터 입력받은 숫자의 팩토리얼을 계산하는 프로그램 작성

```c#
int result = 1;
Console.Write("Enter a number: ");
int num = int.Parse(Console.ReadLine()); //입력받을 숫자를 저장할 변수

for(int i = 1; i <= num; i++)
{
    result *= i;
}

Console.WriteLine("Factorial of " + num + " is " + result);
```
