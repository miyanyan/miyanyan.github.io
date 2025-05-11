---
title: windows自动以管理员身份运行bat文件
description: 转载知乎的一个回答
date: 2023-12-25
categories:
    - windows
tags:
    - windows
    - bat
---

作者：Scruel
链接：<https://www.zhihu.com/question/34541107/answer/243592603>
来源：知乎
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。

正好看到, 那就一行代码解决问题吧

```bat
net file 1>NUL 2>NUL || start "" mshta vbscript:CreateObject("Shell.Application").ShellExecute("cmd.exe","/c pushd ""%~dp0"" && ""%~s0"" %*","","runas",0)(window.close) && exit
```

如果VBScript 被砍掉，则可用 powershell 来执行，原理是一样的：

```bat
net file 1>NUL 2>NUL || powershell Start-Process -FilePath cmd.exe -ArgumentList """/c pushd %~dp0 && %~s0 %*""" -Verb RunAs && exit
```

可支持：

* 带空格的脚本路径
* 自动切换至脚本所在的路径
* 脚本参数传递
* 可隐藏窗口执行脚本（下文）

需要隐藏窗口执行的话，对于 mshta 版本，只需要改变 0 为 1 即可：

```bat
net file 1>NUL 2>NUL || start "" mshta vbscript:CreateObject("Shell.Application").ShellExecute("cmd.exe","/c pushd ""%~dp0"" && ""%~s0"" %*","","runas",1)(window.close) && exit
```

以及 powershell 版本，简单增加 -WindowStyle Hidden 即可：

```bat
net file 1>NUL 2>NUL || powershell Start-Process -FilePath cmd.exe -ArgumentList """/c pushd %~dp0 && %~s0 %*""" -Verb RunAs -WindowStyle Hidden && exit
```

解释一下：

|当前值|当前值含义|
|----|----|
|%~s0|脚本的绝对路径|
|%~dp0|脚本所在目录的绝对路径|
|runas|以管理员权限执行|

使用 cmd.exe 来执行，是为了能让脚本在执行时，自动切换到其所在的路径，因为直接通过 ShellExecute执行，并不能做到自动切换目录。
