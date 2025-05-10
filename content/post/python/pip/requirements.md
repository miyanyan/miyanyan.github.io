---
title: python requirements.txt 里指定操作系统和python版本
description: 在requirements.txt维护可行的版本
date: 2022-11-16
categories:
    - python
tags:
    - python
    - pip
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
---

在给[MetaBCI](https://github.com/TBC-TJU/MetaBCI/blob/master/requirements.txt)编写github ci时发现在不同的平台下依赖不尽相同，版本要求不尽相同，在这里记录下

参考[https://pip.pypa.io/en/stable/reference/requirement-specifiers/](https://pip.pypa.io/en/stable/reference/requirement-specifiers/)

```txt
# ---------------------------------
#  System           platform value
# ---------------------------------
#  AIX              "aix"
#  Linux            "linux"
#  Windows          "win32"
#  Windows/Cygwin   "cygwin"
#  MacOS            "darwin"
# ---------------------------------
atomac==1.1.0; sys_platform == 'darwin'
futures>=3.0.5; python_version < '3.0'
futures>=3.0.5; python_version == '2.6' or python_version=='2.7'
```
