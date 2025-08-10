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

    打开visual studio的CMakeSettings.json，设置为`${env.VCPKG_ROOT}/scripts/buildsystems/vcpkg.cmake`(**语法和vscode不一样!!!**)

    ```json
    {
        "configurations": [{
            "name": "x86-Debug",
            "generator": "Visual Studio 15 2017",
            "configurationType" : "Debug",
            "buildRoot":  "${env.LOCALAPPDATA}\\CMakeBuild\\${workspaceHash}\\build\\${name}",
            "cmakeCommandArgs": "",
            "buildCommandArgs": "-m -v:minimal",
            "variables": [{
            "name": "CMAKE_TOOLCHAIN_FILE",
            "value": "D:\\src\\vcpkg\\scripts\\buildsystems\\vcpkg.cmake"
            }]
        }]
    }
    ```
    参考：https://github.com/MicrosoftDocs/vcpkg-docs/blob/main/vcpkg/examples/installing-and-using-packages.md#cmake
    
2. 配置generator为Ninja
   打开vscode的设置界面，搜索`cmake.generator`，填入`Ninja`即可。
    
