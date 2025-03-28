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
