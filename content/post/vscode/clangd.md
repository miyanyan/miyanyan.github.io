---
title: vscode clangd 设置
description: vscode clangd 插件设置
date: 2025-03-14
categories:
    - vscode
tags:
    - vscode
    - clangd
---

1. 禁用microsoft c++插件的提示
   打开vscode的设置界面，搜索`c++ engine`，选择`C_Cpp.intelliSenseEngine`，设置为`disabled`

   对应的settings.json设置为

   ``` json
   {
    "C_Cpp.intelliSenseEngine": "disabled"
   }
   ```

2. 添加clangd参数
   * 不自动添加头文件
   * 补全时不自动填写函数参数

    打开vscode的设置界面，搜索`clangd arguments`，选择`clangd.arguments`，点击`Add Item`添加对应参数。

   对应的settings.json设置为

   ```json
   {
    "clangd.arguments": [
        "--header-insertion=never",
        "--function-arg-placeholders=0"
    ],
   }
   ```
