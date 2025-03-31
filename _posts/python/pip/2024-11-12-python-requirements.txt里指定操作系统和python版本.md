---
title:      "python requirements.txt 里指定操作系统和python版本"
date:       2024-11-12
categories: [python, pip]
tag: [python, pip]
---


参考[https://pip.pypa.io/en/stable/reference/requirement-specifiers/](https://pip.pypa.io/en/stable/reference/requirement-specifiers/)
```
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
