---
title: vscode cmake 设置
description: vscode cmake 插件设置
date: 2025-07-11
categories:
    - vscode
tags:
    - vscode
    - cmake
---

1. 配置CMAKE_TOOLCHAIN_FILE为vcpkg

   打开vscode的设置界面，搜索`cmake.configureSettings`，点击`Edit in settings.json`并添加：

   ```json
   {
        "cmake.configureSettings": {
            "CMAKE_TOOLCHAIN_FILE": "${env:VCPKG_ROOT}/scripts/buildsystems/vcpkg.cmake"
        },
    }
    ```
