---
title: cython，将py文件编译成pyd/so文件
description: 将python源文件加密的一种方式
date: 2024-11-10
categories:
    - python
tags:
    - python
    - cython
    - pyd
weight: 1       # You can add weight to some posts to override the default sorting (date descending)
---

在windows下使用cythonize可以将py文件编译成pyd，这样就完成了初步的python代码加密工作
使用前需要先安装Cython ```pip install Cython```
然后对py文件执行命令```cythonize -i -3 --directive always_allow_keywords=true xxx.py```
这里有一个要注意的点：

* --directive always_allow_keywords=true. 这一个参数是强制生成keyword，因为有的人在写python文件时，会写成:

  ``` python
  def func(input):
     // do somthing
     return True

  res = func(input=input) # 注意这里，input=input，相当于需要一个keyword
  ```

  但是cython默认不会为一个参数的函数生成keyword，因此当出现上面这种情况时，就会出现py可以运行pyd不可以运行的情况。见[always_allow_keywords](https://cython.readthedocs.io/en/latest/src/userguide/source_files_and_compilation.html)，文档里说关闭always_allow_keywords会slightly faster，但是都用python了这点性能总感觉大差不差的，毕竟只是针对只有一个入参的函数才有用。

下面是一个脚本，用于把目录下的所有py文件转换为pyd

``` python
import os
import sys
import subprocess
from pathlib import Path

def compile_to_pyd(src_path: str, auto_delete: bool):
    if (os.path.isfile(src_path)):
        print("input file")
        py_files = []
        if (src_path.endswith(".py")):
            py_files.append(src_path)
    else:
        print("input dir")
        # 获取目录中的所有.py文件
        py_files_Path = list(Path(src_path).rglob('*.py'))
        py_files = [str(file.absolute()) for file in py_files_Path]
    
    if len(py_files) == 0:
        print("no py files...")
        return
        
    print(f"building {len(py_files)} py files...")
    # 使用Cython编译为.pyd文件
    files_str = " ".join(py_files)
    # --directive always_allow_keywords=true 是为了防止有人写 res = fun(a = a) 这种情况
    # cython默认对于单参数不生成keyword，因此要强制开启
    # https://cython.readthedocs.io/en/latest/src/userguide/source_files_and_compilation.html
    compile_command = f"cythonize -i -3 --directive always_allow_keywords=true {files_str}"
    print(compile_command)
    subprocess.run(compile_command, shell=True, check=True)
    
    if auto_delete:
        # 删除生成的 .py 和 .c 文件
        for py_file in py_files:
            c_file = py_file[:-2] + "c"
            if os.path.exists(c_file):
                os.remove(c_file)
            if os.path.exists(py_file):
                os.remove(py_file)
    

if __name__ == "__main__":

    if len(sys.argv) < 2:
        print("Usage: python compile_to_pyd.py <directory>")
        sys.exit(1)

    dir_name = sys.argv[1]
    auto_delete = "--auto-delete" in sys.argv
    compile_to_pyd(dir_name, auto_delete)

```
