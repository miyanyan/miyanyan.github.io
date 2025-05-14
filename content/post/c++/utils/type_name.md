---
title:      "C++ 编译期获取类名"
description: 给[yalantinglibs]的pr记录
date:       2025-03-31
categories: [c++]
tag: [c++, std]
---

发现[yalantinglibs](https://github.com/alibaba/yalantinglibs/pull/874)的type_string的输出漏掉了const关键字，问了作者发现确实是一个bug，在提pr的过程也发现了这种实现还是有缺陷的，在此记录下

## 原理

* 获取到各个编译器下的原有名字
  * 使用编译器宏获取，msvc下为`__FUNCSIG__`，gcc和clang下为`__PRETTY_FUNCTION__`
  * 使用c++20的source_location的function_name获取，但是注意，**这个函数不是所有的版本的适用**，见[Improve std::source_location::function_name() informativeness #3063](https://github.com/microsoft/STL/issues/3063)

  最终采用宏的方式获取
* 进行拆分截断
  * 创建一个模板函数，模板参数`typename T`，在函数内使用上述的宏得到原始的字符串
  * 三大编译器的输出都是 **前缀 + 内容 + 后缀** 的方式，且前后缀的长度固定，因此找到前缀和后缀的位置并拆分
  * 可以先建立一个模板，我们以`int`为模板，编译期内得到前后缀长度
    * msvc: `class std::basic_string_view<char,struct std::char_traits<char> > __cdecl get_raw_name<int>(void)`
    * gcc: `constexpr std::string_view get_raw_name() [with T = int; std::string_view = std::basic_string_view<char>]`
    * clang: `std::string_view get_raw_name() [T = int]`

**【缺陷】并不能保证各个编译器下的字符串完全一致，包括但不限于多或少一个空格，逗号的位置等等（msvc还会加上class struct union关键字！）**

## 代码

msvc、gcc、clang下均可运行

``` c++
#include <string_view>

template <typename T>
constexpr std::string_view get_raw_name() {
#ifdef _MSC_VER
  return __FUNCSIG__;
#else
  return __PRETTY_FUNCTION__;
#endif
}

template <typename T>
inline constexpr std::string_view type_string() {
  constexpr std::string_view sample = get_raw_name<int>();
  constexpr size_t prefix_length = sample.find("int");
  constexpr size_t suffix_length = sample.size() - prefix_length - 3;

  constexpr std::string_view str = get_raw_name<T>();
  return str.substr(prefix_length, str.size() - prefix_length - suffix_length);
}
```

代码确实很简洁！但是输出不能保证各个编译器的一致性，具体输出见：
[compiler explorer 在线运行链接](https://www.godbolt.org/z/dncsebqKv)
