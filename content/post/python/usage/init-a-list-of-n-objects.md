---
title: python 初始化一个固定大小的list
description: 一个不小心使用碰到的坑
date: 2022-10-26
categories:
    - python
tags:
    - python
    - list
---

* ```AList = [A()] * n```, 此时AList里的所有元素具有相同id, 即是同一个对象，修改AList[0]相当于修改AList的所有元素
* ```AList = [A() for i in range(n)]```, 此时AList里的所有元素都是不同的id, 即是不同的对象, 这样修改任意元素不会影响其他元素
