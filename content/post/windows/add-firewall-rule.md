---
title: windows添加防火墙规则
description: 一个添加防火墙规则脚本
date: 2024-03-06
categories:
    - windows
tags:
    - windows
    - bat
---

bat脚本如下，涉及到的函数主要是 netsh advfirewall firewall add rule 和 netsh advfirewall firewall delete rule

需要注意的一个坑是program的路径不能存在斜杠```/```，需要使用```\```，然而很多软件都会自动把路径生成为类似于`C:/User/...`这种形式，需要在脚本里转换一下

```bat
rem Check if both program path and program name are provided as arguments
if "%~1"=="" (
    echo Usage: %0 program_path program_name
    echo Example: %0 "C:\Path\to\your\program.exe" YourProgramName
    exit /b 1
)
if "%~2"=="" (
    echo Usage: %0 program_path program_name
    echo Example: %0 "C:\Path\to\your\program.exe" YourProgramName
    exit /b 1
)

set "programPath=%~1"
set "programName=%~2"

rem 替换斜杠为反斜杠
set "programPath=%programPath:/=\%"

rem Delete firewall rule
netsh advfirewall firewall delete rule name="%programName%(TCP)"
netsh advfirewall firewall delete rule name="%programName%(UDP)"

rem Add firewall rule for TCP
netsh advfirewall firewall add rule name="%programName%(TCP)" dir=in program="%programPath%" action=allow protocol=TCP

rem Add firewall rule for UDP
netsh advfirewall firewall add rule name="%programName%(UDP)" dir=in program="%programPath%" action=allow protocol=UDP

echo Firewall rules for %programName% added successfully.

```
