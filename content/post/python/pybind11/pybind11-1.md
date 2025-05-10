---
title: pybind11使用(1) cmake + vcpkg 环境配置以及一些坑
description: pybind11使用记录
date: 2022-09-01
categories:
    - python
    - c++
tags:
    - python
    - pybind11
    - c++
    - vcpkg
    - cmake
---

## 看似简单的配置流程

* 安装 : ```vcpkg install pybind11```
* 编写 cmake

  ```cmake
  find_package(pybind11 REQUIRED)

  pybind11_add_module(${PROJECT_NAME} src/main.cpp)
  ```

## 问题随之而来

### [build] LINK : fatal error LNK1104: cannot open file 'optimized.lib'

这个问题在老版vcpkg是存在的，在[某一版本](https://github.com/microsoft/vcpkg/pull/26239)修复
而我的vcpkg好久没pull了，不幸的命中了这个问题，看来用vcpkg得及时更新...

### 无效的函数python_add_library

由于pybind11在python3.8导出的模块只能给对应的python3.8版本用，导致想用多个python版本跑行不通，因此得多个版本编译，cmake脚本应修改为:

```cmake
find_package (Python 3.8 EXACT COMPONENTS Interpreter Development REQUIRED)
find_package(pybind11 REQUIRED)

pybind11_add_module(${PROJECT_NAME} src/main.cpp)
```

**这里有个坑，如果写```find_package (Python 3.8 EXACT REQUIRED)```而不是```find_package (Python 3.8 EXACT COMPONENTS Interpreter Development REQUIRED)```就会提示找不到```python_add_library```**

### python的版本管理

主要有两种方法：

#### 手动安装python版本

下载python并安装，注意要把路径添加到环境变量

#### vcpkg的version特性

使用version特性需要开启[vcpkg的mainfest模式](https://vcpkg.io/en/docs/users/versioning.html)
在CmakeLists.txt的同级目录编写vcpkg.json文件：

```json
{
  "name": "test",
  "builtin-baseline": "c0b6d35a67d2ad358a6ced92b7aad16a7bf17737",
  "dependencies": [
    {
      "name": "python3",
      "version>=": "3.8.3"
    },
    "pybind11"
  ],
  "overrides": [
    {
      "name": "python3",
      "version": "3.8.3"
    }
  ]
}
```

## 终于可以愉快使用pybind11啦
