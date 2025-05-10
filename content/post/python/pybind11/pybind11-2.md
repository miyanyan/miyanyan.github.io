---
title:      "pybind11使用(2) 编写C++类并导出为pyd"
description: pybind11使用记录
date:       2022-09-02
categories: [python, c++]
tags: [c++, pybind11, python, cmake]
---

## 导出的坑

### dynamic module does not define module export function

这是由于导出的包名不一致导致的，需要保证
cmake的project名称

``` cmake
project ("haha" LANGUAGES CXX) # <-- haha

find_package (Python 3.8 EXACT COMPONENTS Interpreter Development REQUIRED)
find_package(pybind11 REQUIRED)

pybind11_add_module(${PROJECT_NAME} src/main.cpp)
```

与
main.cpp里的PYBIND11_MODULE

``` c++
PYBIND11_MODULE(haha, m) { // <-- haha
    m.doc() = "haha c++ wrapper";
}
```

名称一致

### 回调函数

我们可以在cpp里定义一个回调函数，然后python定义一个函数传到c++里，从而从c++取到数据

``` c++
// c++
using PyCallback = std::function<void(pybind11::bytearray)>;

class Haha
{
public:
    void setCallback(PyCallback& pyfn) {
        m_pyfn = pyfn;
    }
    void onDataAvaiable(char* buf, int len) {
        m_pyfn(pybind11::bytearray(buf, len));
    }
private:
    PyCallback m_pyfn;
};

PYBIND11_MODULE(haha, m) {
    pybind11::class_<Haha>(m, "Haha")
        .def("setCallback", &Haha::setCallback);
}

// python
def fn(data):
    print(data)

hahaInstance = m.Haha()
hahaInstance .setCallback(fn)
while True:
    // 阻塞保证hahaInstance一直运行，不断调用callback
```

#### 报错：Some automatic conversions are optional and require extra headers to be included when compiling your pybind11 module

这是由于使用了c++的function但是导出的时候未```#include <pybind11/functional.h>```

#### 报错: DLL load failed while importing xxx

这是由于c++库引用了某个dll，但是没和pyd放到同一路径下，python import 的时候自然找不到对应的dll了

#### 执行python传入的回调函数时直接crash

cpp里不能把PyCallback写成引用

``` c++
class Haha
{
private:
    PyCallback& m_pyfn; // <-- 不能写引用
};
```
