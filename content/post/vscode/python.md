---
title: vscode python 设置
description: vscode python 相关配置
date: 2025-05-30
categories:
    - vscode
tags:
    - vscode
    - python
---

参考[Python settings reference](https://code.visualstudio.com/docs/python/settings-reference)

1. python pyi 文件的搜索目录
pyi文件用于提示IDE自动补全和高亮，在vscode中对应`stubPath`这个参数，它的默认值是`./typings`，也就是根目录的typings文件夹，所以把所有的pyi文件放到该目录就可以使vscode里的python代码自动提示并高亮了
